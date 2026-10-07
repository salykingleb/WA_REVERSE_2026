# parameter_builder — how the /v2/code and /v2/register request parameters are assembled

## Place in the flow

The new Kotlin registration path of WhatsApp 2.26.35.75 (263507522) assembles request parameters in two tiers:

1. **`com/whatsapp/registration/core/http/KotlinRegistrationBridge.java`** — the facade. Each registration HTTP method (generateAuthCode=/v2/code, verifySecurityCode=/v2/register, checkIfExists, fetchAccountDefenceDeviceConfirmation, sendClientFunnelLog, consent) creates an instance of the builder `C40787I4g` via `A02("KotlinRegistrationBridge/<method name>")`, fills in its identifiers and service fields, then passes it to `RetryingHttpClient.A01(...)`.
2. **`X/C40787I4g.java`** (log name **RegistrationRequestBuilder**, loaded from classes9.dex) — a typed "parameter name → string value" builder. Two fields inside:
   - `A00` — a `Map` of parameters (`AbstractC31531as.A0O()` — a LinkedHashMap, preserves insertion order);
   - `A01` — a `Set<String>` of names whose values are **already percent-encoded**.
3. Then `RetryingHttpClient` hands the value map to the native side (`com.whatsapp.wamsys.JniBridge.jvidispatch*` → WCR* builders in libwhatsapp.so), which builds the final query. For keys from the `A01` set the native side does NOT repeat percent-encoding.

Example of an assembly point (KotlinRegistrationBridge.java:1574-1576, generateAuthCodeBlocking method):
```java
A0S(c40787I4gA01, str, str2, str3, str4);      // lg, lc, fdid (A01), expid (A03)
A0T(c40787I4gA01, str5, bArr, bArr2);          // access_session_id (A03), id (A05), backup_token (A05)
c40787I4gA01.A01("token", str8);               // токен как строка
```
Common bridge helpers (KotlinRegistrationBridge.java:1066-1084):
```java
A0R(builder, cc, in)      → A01("cc"), A01("in")
A0S(builder, lg, lc, fdid, expid) → A01("lg"), A01("lc"), A01("fdid"), A03("expid")
A0T(builder, asi, id[], bt[])     → A03("access_session_id", asi) [если != null],
                                    A05("id", bArr), A05("backup_token", bArr2)
```

## Wire format (+LIVE example from the capture)

The builder itself does not go on the wire — it prepares the map. The live key order on the wire is defined by the native side (libwhatsapp.so), not by the insertion order in the LinkedHashMap. Live capture (Samsung SM-A325F, 2026-09-22), /v2/code (50 keys), fragment with the identifiers:
```
method=sms&backup_token=YfOF...&id=Wp1P...&pid=28104&aid=VM60VU78%2FF6NzE1CSRYipwztp4PCM%2F2hlFoIdUZBb4g%3D
&access_session_id=Yrn...&expid=AbCmAYhROweMJWInj2dh_A&in=9206309125&cc=7
&token=DZ3O...%3D&fdid=da1e6667-b5ed-4fa6-9828-53283393c119
```
The `%2F %2B %3D` signs in aid/token are UPPERCASE HEX triplets: this encoding style is set precisely by the `HSJ` class / the native URL encoder with the alphabet `0123456789ABCDEF`.

## How it is formed in the app (builder methods, X/C40787I4g.java)

| Method | Signature | What it does | Who uses it |
|---|---|---|---|
| `A01` | `(String name, String value)` | puts the string AS IS (the only check is `C000800h.A0A(str2, 1)` for null) | fdid, lg, lc, cc, in, token, method, current_screen… |
| `A02` | `(String name, String value)` | puts the string **only if value != null** | advertising_id, login, event_name |
| `A03` | `(String name, String uuidString)` | `UUID.fromString` → `ByteBuffer.allocate(16).putLong(msb)` + `GCM.A1Z(buf, lsb)` → `GCK.A0o(b)` = `Base64.encodeToString(b, 11)` = **URL-safe base64 without padding (RawURL)**; on `IllegalArgumentException` writes the warning `RegistrationRequestBuilder/parseUuidToBytes/invalid UUID format` and does NOT put the key | **expid**, **access_session_id** |
| `A04` | `(String name, byte[] b)` | bytes → `GCK.A0o` = base64 RawURL (flags 11) | authkey, e_ident, e_keytype, e_regid, e_skey_id, e_skey_val, e_skey_sig |
| `A05` | `(String name, byte[] b)` | bytes → `HSJ.A00(b)` (RFC3986 percent-encode) **and** the name is added to the "already encoded" Set `A01` | **id**, **backup_token** |
| `A06` | `(Map<String, byte[]> map)` | for each pair: `A00.put(key, HSJ.A00(value))` + `A01.add(key)` — batch percent-encode of byte[] values (the native blobs `_gg`, `gpia`, `_gi`, `_ga`, `_gs`, `client_metrics`) | integrity/gpia parameters |
| `A00` | `(String name, int i)` | boolean helper: 0 → `"false"`, 1 → `"true"`, otherwise puts nothing | flags like prefer_sms_over_flash |

The `Base64.encodeToString(..., 11)` flags = `URL_SAFE(8) | NO_WRAP(2) | NO_PADDING(1)` — 22 characters
per 16 bytes, no `=`, alphabet `A-Za-z0-9-_`.

**Persistence:** the builder itself is a one-off per-request object (created by `A02(...)`), it stores
nothing. All the value generators (C00L.A0G, IF6.A0u/A0t, C1Ih, C1Ik, C12040h1) live in their own
stores, see the separate chapters.

## Native implementation

- `RetryingHttpClient.A01(builder, ...)` serializes the `builder.A00` map and the `builder.A01`
  set into a `JniBridge.jvidispatch*` call. In libwhatsapp.so (memory dump, load base 0x7cfd633000)
  the /v2 map dispatcher on the `0x7cfdd2c640` page lays out the structure fields into its own
  sid keys: `[x1+0x78]→sid 102 "aid"`, `[x1+0x58]→"_gi"`, `[x1+0x60]→"_gp"`, `[x1+0x68]→"_ga"`,
  `[x1+0x70]→"_ge"`, `[x1+0x80]→"_gs"` (obfuscated string table, AES-ECB).
- The final query and its key ORDER are built by the native WCR builder — that is why the
  on-the-wire order (method, backup_token, _gs, sim_mnc, id, mnc, _gg, …) does not match the Java
  insertion order.
- The "already encoded" set (`builder.A01`) is passed to the native side as is: for the keys
  id/backup_token/*-blobs repeated URL-encoding is skipped. The point: HSJ has already produced
  the final wire form, encoding again would give double escaping (`%2F` → `%252F`).

## Value selection conditions (variants, ranges, when absent)

- `A02` — the only path where a key can be ABSENT on the wire (a null value): this is how
  `advertising_id` disappears (EU region, limit ad tracking, no GMS).
- `A03` silently skips the key on an invalid UUID (a parse error) — but in the live flow the UUID
  is always valid, because it comes from the generators.
- `A0T` puts `access_session_id` only if the argument != null.
- `A00(int)` silently ignores values outside {0,1}.
- The full native list of registration parameter names (rodata libwhatsapp.so @0x2f35e0+):
  id, backup_token, fdid, expid, access_session_id, token, advertising_id, authkey, e_ident,
  e_keytype, e_regid, e_skey_id, e_skey_val, e_skey_sig and others. Separately there exists a
  native key `device_exp_id` (next to expid, not filled in on the Java path).

## Cryptography and encoding

- **HSJ.A00 (X/HSJ.java:8-46)** — a byte-by-byte RFC3986 percent-encoder: only `A-Za-z0-9 - . _ ~`
  pass unencoded; everything else → `%XX` in UPPERCASE HEX. The HEX alphabet is
  `C40787I4g.A02 = "0123456789ABCDEF".toCharArray()` (X/C40787I4g.java:47-51). The buffer is
  allocated by `GCK.A0q(len*3)` — the worst case of 3 characters per byte.
- **GCK.A0o (X/GCK.java:273)** — `Base64.encodeToString(b, 11)` = RawURL-base64.
- In A01/A02 the strings go out raw — the native side encodes them (the style is the same: `%XX`
  UPPERCASE).

## Examples

Live values (Samsung SM-A325F, 2026-09-22, pid=28104) passing through each method:
- A01: `fdid=da1e6667-b5ed-4fa6-9828-53283393c119`, `cc=7`, `in=9206309125`, `lg=ru`, `lc=RU`;
- A03: `expid=AbCmAYhROweMJWInj2dh_A` (22 chars, no `=`);
- A04: `authkey=rsgyL8y0zyOXIJT4KofOJGJaxrlKUp6TuBsPvGzZAzs` (43 chars RawURL from 32 bytes);
- A05: `id` and `backup_token` — percent-encoded strings of 20..60 characters from 20 random bytes;
- A06: the blobs `_gs`, `_gg`, `gpia`, `_gi`, `_ga`, `client_metrics` — long percent-strings.
