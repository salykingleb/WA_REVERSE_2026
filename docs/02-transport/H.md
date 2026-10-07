# H — request signature (ECDSA-P256-SHA256 over the ENC string)

## Place in the flow

`H` — the second component of the POST request body for `/v2/code` and `/v2/register`:

```
ENC=<конверт>&H=<base64.RawURL(DER-подпись)>
```

Purpose: to prove to the server that ENC was created by the holder of the private key
whose certificate is attached in the `Authorization` header (the keystore attestation chain).
The "H + Authorization" pair is the key-attestation mechanism of registration: the server
extracts the public key from the leaf certificate, verifies the H signature with it, and
at the same time validates the chain up to the Google root + the attestation
extension KeyDescription (see Authorization.md).

Assembly — `RetryingHttpClient.A00`, log tag `RegistrationBodyBuilder/signWithKeyAttestation`
(dexdump classes9.dump, offsets 0x0263–0x02b6).

## Wire format (type, encoding, length, LIVE example from the capture)

- Type: a form-urlencoded value of the body parameter `H`.
- Signed data: the ASCII bytes of the `ENC` STRING (the base64url value itself, including
  `pub`, ciphertext and tag; NOT a hash passed separately, and NOT the decrypted query).
- Algorithm: `SHA256withECDSA` on an EC P-256 key — inside JCA the signature is computed as
  ECDSA over SHA-256(ASCII(ENC)); the output is **DER** (ASN.1 SEQUENCE { INTEGER r, INTEGER s }).
- Encoding: `X/GCK.A0o` = `Base64.encodeToString(sig, 11)` = **base64.RawURL** (no `=`).
- Length: a P-256 DER signature — 8–72 bytes, typically ~70–72 bytes → b64url ~94–96 characters.
- The live H value is absent in the text dump (the dump contains only the URL);
  the format was reconstructed from the dexdump code and the reference Frida comment on the live body.

## How it is built in the app (Java/Kotlin class file:line, code fragments)

The call chain in RetryingHttpClient.A00:

1. Input: `B0W.A1Z(encValue)` — the ASCII bytes of the ENC string.
2. `X/C13D.A07(payload, challengeBytes)` — the "blacknoise" class
   (C13D.java:577–613), verbatim:
   ```java
   public byte[] A07(byte[] bArr, byte[] bArr2) {
       if (!A06()) return null;                     // AB-флаг 1934: аттестация включена?
       A02(C02S.A01, bArr2);                        // гарантирует свежую пару в Keystore
       KeyStore keyStore = KeyStore.getInstance("AndroidKeyStore");
       keyStore.load(null);
       KeyStore.Entry entry = keyStore.getEntry(A01(), null);   // alias static
       if (entry instanceof KeyStore.PrivateKeyEntry) {
           Signature signature = Signature.getInstance(this.A01.A0f(2075)); // для EC = SHA256withECDSA
           signature.initSign(((KeyStore.PrivateKeyEntry) entry).getPrivateKey());
           signature.update(bArr);                  // update(ASCII(ENC))
           bArrSign = signature.sign();              // DER-выход
       }
       ...
       return bArrSign;
   }
   ```
3. `X/GCK.A0o(sig)` (GCK.java:273) → base64.RawURL → the H value.
4. Body: `HWs.A02("ENC") + "=" + enc + "&" + HWs.A03("H") + "=" + h`
   (`X/AbstractC39157HWs.java`; the "ENC"/"H" literals — the XOR-18 pool `X/AbstractC40561Hxd.A05/A09`).

**Key pair generation (leaf)** — `C13D.A02` (C13D.java:86–210), called before
the signature if the pair is missing or stale:

```java
KeyPairGenerator kpg = KeyPairGenerator.getInstance("EC", "AndroidKeyStore"); // A0f(2076), default "EC"
KeyGenParameterSpec.Builder b = new KeyGenParameterSpec.Builder(alias, 4)      // :177, 4 = PURPOSE_SIGN
        .setDigests("SHA-256", "SHA-512")
        .setUserAuthenticationRequired(false)
        .setCertificateNotAfter(date);                                        // now + A0Y(2079) сек
if (AnonymousClass075.A00()) {                                                // :181–194 — аттестация
    ...
    long unix = C08A.A00(clock) / 1000;
    ByteBuffer bb = ByteBuffer.allocate(challenge.length + 8 + 1);
    bb.order(ByteOrder.BIG_ENDIAN);
    bb.putLong(unix);                                                          // 8B BE времени
    bb.put((byte) 31);                                                         // 0x1F
    bb.put(challenge);                                                         // байты challenge
    b.setAttestationChallenge(bb.array());                                     // → KeyDescription.challenge
}
kpg.initialize(b.build());
kpg.generateKeyPair();
```

**The challenge on generation** (C13D.A02:186–193): in registration calls challenge =
the client's public authkey, 32 bytes (`X/AnonymousClass139.A0I()`; in `X/DVW.java:166–174`
the parameter is literally named "Client Public Key for Attestation"). Structure:

```
challenge = putLong_BE(unixSeconds) || 0x1F || authkey(32B)
          = 00000000 || uint32_BE(unixSeconds) || 0x1F || authkey   (до 2106 г.)
```

The first 4 zero bytes are a consequence of the Java `putLong` (an 8-byte BE long; unixtime
values < 2^32 give a zero high half-block).

## Native implementation (if involved)

The signature itself is performed by Android Keystore (the device's TEE/keymaster) — the private
key never leaves the TEE. The Java layer passes only the aliases into the native wamsys path
via `X/AbstractC39157HWs` (`A00` → provider C13D, id 2874; `A01` → the keystore
provider, id 6015). No `_WCAPI*` functions are involved in the H path — the operation
is fully JCA (`Signature.getInstance` / `KeyStore.getInstance("AndroidKeyStore")`).

## Conditions for choosing values (variants, ranges, when absent)

- **H is absent** if attestation is off: `C13D.A06()` = AB flag 1934
  (C13D.java:573–575) — when false, A07 returns null and the body stays `ENC=…`.
- **Key freshness (leaf)**: the app uses the alias `_static`
  (`my_personal_mini_pony_static` + an optional suffix) with PERIODIC refresh:
  ```java
  j = prefs.getLong("ka_static_refresh_ts", 0);       // C13D.java:116
  interval = A00.A0Y(4878);                            // AB-флаг 4878, секунды
  if (now >= j + interval) { /* регенерация пары с новым challenge */ }
  ```
  That is, one and the same leaf key signs MANY requests within the interval, rather than
  "a new key per request". The dynamic alias (`my_personal_mini_pony`) has
  its own timer `ka_refresh_ts` / AB 2079 and does not take part in the registration signature.
- **Signature algorithm**: the string from AB config id 2075; for an EC key it is SHA256withECDSA
  (for the RSA branch — PKCS1, but registration uses EC).
- Alias suffix: on regeneration `ka_key_store_static_alias_suffix`
  = `UUID.randomUUID().toString()` (C13D.java:150–163) may be written — the alias becomes
  `my_personal_mini_pony_static_<uuid>`; wiping all pairs — `C13D.A04()` (reset
  of `ka_*_ts` and the suffixes).
- Key digests: setDigests("SHA-256","SHA-512") — in KeyDescription this is
  digests = {4 (SHA-2-256), 6 (SHA-2-512)}.

## Cryptography and encoding

- Formula: `H = base64.RawURL( DER( ECDSA-P256-SHA256( leafPriv, SHA256( ASCII(ENC) ) ) ) )`.
  JCA `SHA256withECDSA` hashes the input itself: the ASCII bytes of the
  ENC string are fed into update(), the SHA-256 digest is computed inside; a deterministic
  signature form is not guaranteed (standard ECDSA with a random k from the TEE).
- DER: `SEQUENCE { r INTEGER, s INTEGER }` — Java `Signature.sign()` for EC
  returns ASN.1 DER (not raw 64 bytes r||s).
- base64: flag 11 = URL_SAFE(8) | NO_WRAP(2) | NO_PADDING(1) — the alphabet
  `A-Za-z0-9-_`, no `=`.
- The private leaf key: Android Keystore, EC P-256, PURPOSE_SIGN, TEE-backed;
  the corresponding certificate (with the attestation extension) goes into Authorization.

## Examples

The body scheme with H (values of the proportions of the live request):

```
ENC=K7Tz…<~2.5КБ base64url>&H=MEUCIQD…<~94-96 символов base64url DER-подписи>
```

The challenge structure by which the server can link the leaf to a specific client
(live authkey of the 2026-09-22 session, base64url → 32B):

```
authkey = rsgyL8y0zyOXIJT4KofOJGJaxrlKUp6TuBsPvGzZAzs
challenge = 00 00 00 00 || BE32(0x68F2xxxx ~ unix 2026-09-22) || 1F || <32B authkey raw>
итого 45 байт → попадает в расширение KeyDescription leaf-сертификата (1.3.6.1.4.1.11129.2.1.17)
```

Reproduction check: VerifyASN1(leafPub, SHA256(ASCII(ENC)), DER) == true;
leafPub is taken from the first certificate of the Authorization chain (the order root→…→leaf,
the leaf is the last).
