# ENC — encryption envelope of all registration parameters

## Place in the flow

`ENC` — the first component of the POST request body for `/v2/code` and `/v2/register`:

```
ENC=<base64.RawURL(ephemeralPub32 || AES-256-GCM-ciphertext || GCM-tag)>&H=<signature>
```

Inside the envelope lies THE SAME percent-encoded query string that goes in plaintext
into the URL after `?` (double submission of parameters — see
http-transport.md). That is, the plaintext part serves for compatibility/logging,
while ENC is the protected copy, subsequently signed via `H`. The body is assembled in
`RetryingHttpClient.A00` (log tag `RegistrationEncryption/encryptQueryString`,
dexdump classes9.dump, offsets 0x01f4–0x0249).

## Wire format (type, encoding, length, LIVE example from the capture)

- Type: a form-urlencoded value of the body parameter `ENC`.
- Encoding: `Base64.encodeToString(bytes, 11)` = URL_SAFE | NO_WRAP | NO_PADDING
  = **base64.RawURL** (alphabet `A-Za-z0-9-_`, no `=`, no line breaks).
- Byte structure before base64:
  ```
  [0..31]   ephemeralPub — 32 raw-байта публичного ключа X25519 (БЕЗ байта типа 0x05)
  [32..]    ciphertext||tag — AES-256-GCM: шифртекст длины len(plaintext) + 16 байт GCM-тега
  ```
- Length: `len(ENC_b64) = ceil((32 + len(query) + 16)/3)*4 (without padding)`. For the live
  /v2/code query ~1.5–2 KB → ENC ~2–2.7 KB of base64url characters.
- The live body is absent in the text dump of the capture (the dump contains only the URL),
  but the fact "URL query == plaintext ENC" for /v2/code is confirmed by the structure of the
  app code: one and the same query variable goes both into the URL and into the encryption
  input. A related live example of the same scheme — the `_gs` parameter (the `{"em": …}` envelope
  is encrypted with the same X25519 server key and AES-256-GCM with a zero IV): the live `em` = 167 bytes
  decoded = 32B ephemeralPub + 135B ct (plaintext 119B before the tag) — the same
  `pub||ct||tag` structure.

## How it is built in the app (Java/Kotlin class file:line, code fragments)

The algorithm per the dexdump of RetryingHttpClient.A00 (tag `RegistrationEncryption/encryptQueryString`):

1. **Plaintext**: `join("&", request.A00.entrySet, mapper IuA-46)` — the same `k=v` pairs
   as in the URL (see params-order.md about the mapper encoding).
2. **Ephemeral X25519 pair**: `X/AbstractC25231B8l.A01()` (CryptoUtils, classes7.dex):
   ```java
   public static final C25243B8x A01() {
       C1KL c1kl = C1KK.A00("best").A00;          // провайдер whispersystems "best"
       byte[] priv = c1kl.generatePrivateKey();    // генерация + clamping (см. ниже)
       byte[] pub  = c1kl.generatePublicKey(priv);
       ...
       return new C25243B8x(new C25244B8y(priv), new C25226B8g(pub, (byte)5)); // тип 5 = Djb/X25519
   }
   ```
   (AbstractC25231B8l.java:88–96). Key type 5 — "DJB" (Curve25519), libsignal-style
   key serialization `[0x05 || 32 bytes]`.
3. **ECDH with the server key**: `B8l.A09(priv, B8g(HV4.A00, 5))`:
   ```java
   public static final byte[] A09(C25244B8y priv, C25226B8g pub) {
       return C1KK.A00("best").A02(pub.A01, priv.A00);  // calculateAgreement — raw shared secret
   }
   ```
   (AbstractC25231B8l.java:102–106). **Without any KDF** — the shared secret is used directly further on.
4. **AES-256-GCM**:
   - cipher: `X/GCL.A13()` (GCL.java:150–151) = `Cipher.getInstance("AES/GCM/NoPadding")`;
   - key: `X/B0Y.A1I(shared)` (B0Y.java:217) = `new SecretKeySpec(shared, "AES")` — 32 raw bytes of shared;
   - parameter: `new GCMParameterSpec(128, new byte[12])` — **IV = 12 zero bytes**, tag 128 bits, **no AAD**;
   - encryption: `X/AbstractC25020B0a.A1W(key, spec, cipher, plain, 1)` = `init(ENCRYPT_MODE)` + `doFinal` → `ct||tag`.
5. **Public key prefix**: `copyOfRange(pub.A00(), 1, 33)` — the `[0x05||pub32]` serialization
   is trimmed to 32 raw bytes (the leading type byte 0x05 is discarded),
   then `concat(pub32, ct)`.
6. **Encoding**: `X/GCK.A0o(bytes)` (GCK.java:273) =
   `Base64.encodeToString(bytes, 11)` = base64.RawURL.
7. **Fallback**: on an encryption exception the body is sent with the plaintext
   (log `buildPostBody/encryption failed, using plain query string`).

## Native implementation (if involved)

The encryption itself is performed in the Java layer (Kotlin path). In the wamsys path the envelope
is built by the native libs.so code (obfuscated, reconstructed from the bridge dexdump and
the live wire — the wire format is identical). In the native library the string
literals are obfuscated; the server key and the scheme are confirmed by the Java side
(`X/HV4`, `X/AbstractC25231B8l`) and the live sizes of the `_gs` envelope.

## Conditions for choosing values (variants, ranges, when absent)

- The ephemeral pair is generated **anew for each request** (no cache): /v2/code and
  /v2/register have different ephemeralPub and different ciphertexts.
- The server key is the single constant of the scheme (see below), it does not depend on the request.
- The IV is always 12 zero bytes — uniqueness/confidentiality is provided
  only by the ephemerality of the key (each ciphertext under a new key).
- AAD is not used.
- ENC is absent only in the fallback branch on an encryption failure (the client sends the
  plaintext query in the body); in a normal live capture it is always present.
- The 0x05 prefix byte is NOT transmitted — exactly 32 bytes of pub go into ENC.

## Cryptography and encoding

- **Key class**: X25519 (Montgomery ladder / DJB Curve25519), private key
  32 bytes with standard clamping before use:
  ```
  priv[0]  &= 248;      // сброс младших 3 бит
  priv[31] &= 127;      // сброс старшего бита
  priv[31] |= 64;       // установка бита 254
  ```
  (clamping is performed inside `generatePrivateKey()` of the "best" provider —
  whispersystems/curve25519-java; the result conforms to RFC 7748).
- **Server key** (hardcoded in the app): `X/HV4.java:5`:
  ```java
  public static final byte[] A00 = {-114,-116,15,116,-61,-21,-59,-41,-90,-122,92,108,60,-124,56,86,-80,97,33,-52,-24,-22,119,77,34,-5,111,18,37,18,48,45};
  ```
  In hex (Java signed byte → unsigned): **8e8c0f74c3ebc5d7a6865c6c3c843856b06121cce8ea774d22fb6f122512302d**.
  This is the same key used by the `_gs` envelope (the single WhatsApp registration
  server key).
- **ECDH**: `shared = X25519(ephemeralPriv, serverPub)` — 32 bytes.
- **KDF: NONE.** The raw shared secret immediately becomes the AES-256 key (SecretKeySpec
  "AES"); HKDF/SHA hashing is not applied.
- **Cipher**: AES-256-GCM, IV = `00*12`, tagLength = 128 bits, AAD = null.
- **Output**: `base64.RawURL(pub32 || ciphertext || tag)`.

The formula as a whole:

```
priv = random32(); clamp(priv)
pub  = X25519_base(priv)                     // 32B
shared = X25519(priv, 8e8c0f74…302d)          // 32B
(ct, tag) = AES-256-GCM(shared, IV=0^12, plaintext=query, AAD=∅)
ENC = b64url_nopad(pub ‖ ct ‖ tag)
```

## Examples

A schematic view of the body (real proportions of the live /v2/code request, plaintext =
the string of 50 parameters `method=sms&backup_token=…&…&lc=RU`):

```
ENC=AdwPdMMXq2Tj…<32B pub как base64url, первые 43 символа без паддинга cover 32 байта>…<ciphertext>…&H=<…>
```

A verification example of the structure from the live related `_gs` envelope of the same session:
Live capture: `em` decoded = 167B = 32B ephPub + 135B ct with plaintext 119B
(119 + 16 tag = 135) — exactly the same `pub||ct||tag` formula, the same server key.

Self-check on reproduction: the length of the ENC b64 string without "=" ≈ (48 + the
query length + 16) * 4/3; the first 43 characters decode into 32 bytes of pub (there is no 0x05 byte);
GCM-decrypt with the zero IV and the X25519(priv) key yields the original query string.
