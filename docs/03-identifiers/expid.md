# expid — device ID for telemetry, UUID v3/v4 → 16 bytes → base64 RawURL

## Place in the flow

`expid` (performance device id) — a long-lived device-installation identifier derived from
`Settings.Secure.ANDROID_ID`. One per installation, survives restarts and (on Android 8+)
uninstallation. It is sent in `/v2/code`, `/v2/register`, funnel-log, pre-PN logs,
account-defence. The call point — `IF6.A0G(if6, strA0C)` → `C018208l.A0J().A03()` (the call in
`RequestCodeRepository$requestCode$2.java:354/412`), then as the 5th argument into
`KotlinRegistrationBridge.A0S(builder, lg, lc, fdid, expid)` → `C40787I4g.A03("expid", str4)`
(KotlinRegistrationBridge.java:1071-1076).

## Wire format (+LIVE example from the capture)

A UUID string → 16 raw bytes (msb+lsb big-endian) → URL-safe base64 without padding: 22
characters, no `=`. The UUID version/variant bits are **preserved in the bytes** — on the wire
one can see whether it is v3 or v4.

LIVE example (Samsung SM-A325F, android_id=154866fd3b2cf260, capture of 2026-09-22):
```
expid=AbCmAYhROweMJWInj2dh_A
```
Decode: `01b0a601-8851-3b07-8c25-62278f6761fc` → **version=3** (the nibble `3`),
variant=RFC4122.
The value is IDENTICAL in `/v2/code` and `/v2/register`.

## How it is formed in the app

**Generator/store — `X/C1Ik.A03()` (BlockStoreDeviceIdStore — the class name is FALSE for this
build, Block Store is not involved here):**
```java
public final String A03() {
    String string = A02().getString("perf_device_id", null);      // 1) уже в SP → вернуть
    if (string == null) {
        C0AP c0apA0O = ((C0AO)this.A00.A00.get()).A0O();          // 2) ContentResolver
        if (c0apA0O == null || (strA01 = C00L.A01(c0apA0O)) == null || C0C7.A0p(strA01)) {
            string = UUID.randomUUID().toString();                //    ветка v4: сломанное окружение
        } else {
            byte[] bytes = strA01.getBytes(C07k.A05);             //    UTF-8 байты ANDROID_ID
            string = UUID.nameUUIDFromBytes(bytes).toString();    //    ветка v3: MD5 name-based
        }
        A04(string);                                              // 3) запись в SP (один раз)
    }
    return string;
}
```
- `C00L.A01(c0ap)` (X/C00L.java:556-559): `Settings.Secure.getString(resolver, "android_id")`.
- `C0C7.A0p(str)` — the "all characters are whitespace" check (blank).
- `UUID.nameUUIDFromBytes` = UUID **v3**: MD5(name), then `b6=(b6&0x0f)|0x30`,
  `b8=(b8&0x3f)|0x80` (the standard java.util.UUID semantics).

**Write — `A04` (X/C1Ik.java:44-58):** if `perf_device_id` is not in SP — write it and log
`BlockStoreDeviceIdStore/SP.initPerfDeviceId/wrote/thread=…`; if it is there —
`BlockStoreDeviceIdStore/SP.initPerfDeviceId/noop-sp-already-set` (the value is not overwritten).

**Persistence:** SharedPreferences `perf_device_id`. Written once per installation. On
reinstallation on Android 8+ ANDROID_ID survives the uninstall (tied to the signing key + the
user) → regeneration yields THE SAME v3 (MD5 is deterministic) → expid is in fact unchanged
until a factory reset. It changes: factory reset, user switch, Android <8 (android_id could be
null/change).

## Native implementation

The value is Java; in the builder `C40787I4g.A03` the UUID string is parsed (`UUID.fromString`)
and put as base64 RawURL. The native parameter table (rodata libwhatsapp.so @0x2f35e0+) contains
both `expid` and a SEPARATE `device_exp_id` — the latter is not filled in on the Java
registration path.

## Value selection conditions (variants, ranges, when absent)

- **v3** (`UUID.nameUUIDFromBytes(android_id.getBytes(UTF-8))`) — the main branch: the
  ContentResolver is available AND `android_id` is not null AND not blank. On Android 8+
  ANDROID_ID is guaranteed to exist (16 hex characters, 64 bits, per-user+signing-key) → on real
  devices **~99% of registrations give v3**, regardless of the presence of Google Play Services
  (Huawei/emulators — also v3).
- **v4** (`UUID.randomUUID()`) — the broken-environment branch: a null ContentResolver, a null or
  blank android_id (old ROMs, exotic VMs without a settings-provider). In the wild ≤1-2%.
- UUID versions ∈ {3, 4}; **never v1** — the version nibble is distinguishable on the wire.
- The parameter is always sent (A03 with a valid UUID always puts the key).

## Cryptography and encoding

- v3: MD5 of the UTF-8 bytes of the ANDROID_ID string (not a hash "for security", but the
  standard name-based UUID derivation).
- v4: 122 random bits of `UUID.randomUUID()`.
- Encoding: `C40787I4g.A03` → `ByteBuffer.allocate(16).putLong(uuid.getMostSignificantBits())`
  + `GCM.A1Z(buf, uuid.getLeastSignificantBits())` → `GCK.A0o(b)` =
  `Base64.encodeToString(b, 11)` — URL_SAFE|NO_WRAP|NO_PADDING. The length is always 22
  characters.

## Examples

- Live value: `AbCmAYhROweMJWInj2dh_A` = 16 bytes
  `01 b0 a6 01 88 51 3b 07 8c 25 62 27 8f 67 61 fc`;
  byte 6 (0-based) = `3b` → version 3; byte 8 = `8c` → variant 10xx.
- Formula of the v3 branch: `expid_uuid = nameUUIDFromBytes("154866fd3b2cf260".getBytes(UTF-8))`.
- Wire position (the native order of the code request): between `in` and `simnum`.
