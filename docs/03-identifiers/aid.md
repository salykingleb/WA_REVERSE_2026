# aid — native device identifier: a SHA-256 chain from ANDROID_ID in base64

## Place in the flow

`aid` — an identifier added by the **native side** (libwhatsapp.so) when building the query of
`/v2/code` and `/v2/register`. In the Java code (jadx) the literal `"aid"` is absent — the
parameter appears only in the native builder, therefore pure static analysis over smali/jadx does
not "see" it. The live capture of 2026-09-22 (Samsung SM-A325F) proved the parameter's presence
in both requests: **aid is on the wire**, the value is stable between code and register and
between process launches (including a phone reboot). The native parameter map is returned to Java
as a `Map<String,byte[]>` (the value = an ASCII base64 string) and gets into the query by the
jvidispatch engine.

## Wire format (+LIVE example from the capture)

A percent-encoded string of a standard base64 of 32 bytes: the raw material is 44 characters of
base64-std **with `=`** (the 44th character is `=`), on the wire it is escaped as `%2F`, `%2B`,
`%3D` (UPPERCASE HEX).

LIVE example (Samsung SM-A325F, android_id=154866fd3b2cf260):
```
aid=VM60VU78%2FF6NzE1CSRYipwztp4PCM%2F2hlFoIdUZBb4g%3D
```
(before encoding: `VM60VU78/F6NzE1CSRYipwztp4PCM/2hlFoIdUZBb4g=`; hex of the 32 bytes:
`54ceb4554efcfc5e8dcc4d42491622a70ceda783c233fda1945a087546416f88`).

## How it is formed in the app (the native chain, libwhatsapp.so)

The full chain is proven by the disassembler (the dump's load base 0x7cfd633000):

1. **Obfuscated string table** (AES-ECB): sid 102 = `"aid"`, sid 60 = `"android_id"`,
   sid 57-59 = `Settings$Secure` / `getString` / the signature.
2. **The /v2 map dispatcher** (`0x7cfdd2c640…0x7cfdd2c6cc`): each structure field → its own key:
   `[x1+0x58]→"_gi"`, `[x1+0x60]→"_gp"`, `[x1+0x68]→"_ga"`, `[x1+0x70]→"_ge"`,
   **`[x1+0x78]→"aid"`**, `[x1+0x80]→"_gs"`.
3. **Filling the aid field** (`0x7cfdd2c56c-0x7cfdd2c584`): `ldr x8,[x19,#0x78]; cbnz → skip`
   — a cache on the structure (this is why the value is the same in all the session's requests);
   otherwise the getter `0x7cfdd3145c` → the setter `0x7cfdd42d38`.
4. **The getter** `0x7cfdd3145c` — pure JNI: `currentActivityThread` → Context →
   `getContentResolver` → `FindClass android/provider/Settings$Secure` →
   `GetStaticMethodID getString` → `CallStaticObjectMethod(…, "android_id")` → `GetStringUTFChars`
   → std::string. I.e. the source is **`Settings.Secure.getString(resolver, "android_id")`**.
5. **The transformation** `0x7cfdd2ef1c(std::string)`:
   - `0x7cfdd71da8` — **SHA-256** (WESCryptoProviderSha256Context, a 104B context; the IV is the
     standard `6a09e667 bb67ae85 3c6ef372 a54ff53a 510e527f 9b05688c 1f83d9ab 5be0cd19`, the
     K-table is standard, update/final canonical) → a 32B digest;
   - f1 `0x7d0270ba90` (merged, Rust) — **base64 encoding with the standard alphabet** (the
     clusters `and w?,#0x3f`, the std alphabet in rodata) → 44 characters;
   - f2 `0x7d029abe6c` — buffer → std::string.

**Persistence: NONE.** The raw 32 bytes and the aid base64 string were not found by a binary grep
either in `/data/data/com.whatsapp` (the whole directory), or in `/sdcard/Android/data/com.whatsapp`,
or in the keystore — the value is **recomputed at runtime** on every construction of the request
map and lives only in the structure field `+0x78` (a cache for the duration of the session /
process). Therefore, with an unchanged android_id the value is stable for arbitrarily long, and
when the android_id changes (factory reset, user switch) — it will change along with it.

## Native implementation

See above — the parameter is entirely native. Key addresses (dump base 0x7cfd633000):
getString `0x7cfdd76a64` (off 0x6a464); the field dispatcher `0x7cfdd2c640`;
cache/getter/setter `0x7cfdd2c56c` / `0x7cfdd3145c` (off 0x69e45c) / `0x7cfdd42d38`; transform
`0x7cfdd2ef1c` (off 0x6bc1c); the sha256 wrapper `0x7cfdd71da8`; the b64 loop merged
`0x7d0270bb40-0x7d0270c480`; the std alphabet `0x7d025a2e7c / 0x7d025ad7b9 / 0x7d025d3a30`. The
"logging" duplicate builder `0x7cfdd402c0+` puts the same pair (sid 0x66 + the getter) next to
`_ge/_ln/_iln/_lh/_dh` — a different usage site, the value is the same.

## Value selection conditions (variants, ranges, when absent)

- Always exactly 32 bytes → 44 characters of base64-std, a trailing `=`.
- Sent in code and register always when android_id is available; an empty/unavailable android_id
  would give an empty value (the branch was not observed on a live device).
- Stability: one installation = one value (even after a phone reboot — verified on ≥3 process
  launches); changes only together with android_id.
- In the decrypted integrity blobs (gpia/_gi/_gg/_ga/_gs) there is NO `aid` field — it is a
  standalone query parameter; the closest "relative" in _gi is the key `did` (build display id),
  which has nothing to do with aid.

## Cryptography and encoding

- The core (the structure is confirmed): SHA-256 of the android_id string (`Settings.Secure`,
  sid 60) → base64-std, 44 characters with `=`. There are no other data sources in the pipeline —
  the formula `b64(sha256(android_id))`.
- The remaining unknown — a salt step inside the Rust chain f1 (the region
  0x7d0270b000-0x7d02710000): the bare formula gives
  `b64std(sha256("154866fd3b2cf260")) = 2epZwjZwH27…` ≠ the live reference `VM60VU78/…` (a
  brute-force of ~7.6k combinations: prefixes/suffixes, HMAC, double hashes, raw android_id
  bytes, GSF/did identifiers — no matches). I.e. the input composition is established, the exact
  byte-by-byte assembly of the hashed buffer is not.
- Wire encoding: percent-encode UPPERCASE (`%2F`, `%3D`), the style of HSJ / the native URL
  encoder.

## Examples

- Live: `aid=VM60VU78%2FF6NzE1CSRYipwztp4PCM%2F2hlFoIdUZBb4g%3D` (code and register are
  identical).
- Heap confirmation: the aid base64 string was found @0x7b92500868 in a native malloc region next
  to JNI strings and the path `registration…VerifyPhoneNumber.xml` — the context of the native
  builder.
- Device: android_id=154866fd3b2cf260 (settings_secure.xml), GSF id=3676109938468977878,
  em.did=200cde4cb48d2b11 — none of them directly equals the formula's input (the salt is
  unknown).
