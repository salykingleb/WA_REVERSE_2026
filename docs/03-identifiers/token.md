# token — code request anti-fraud token: HMAC-SHA1(secretKey, certDER || md5Hash || phone)

## Place in the flow

`token` — an "app+device+number" signature with which the server verifies that the code request
was sent by a genuine APK. It is computed by the Java function `X/I0U.A01(context, phoneNumber)`
(the call: `HV2.A00.A01(c39934Hme.A00, str3)` in RequestCodeRepository$requestCode$2.java:165 —
the national number at the input). It is put into the builder as a plain string
`C40787I4g.A01("token", str8)` (KotlinRegistrationBridge.java:1576, 235/350 — account-defence).
In the live capture it is present **only in `/v2/code`** (in `/v2/register` there is no token
parameter — per the live capture data, the "Only in code" list); account-defence requests also
carry the token.

## Wire format (+LIVE example from the capture)

20 bytes of HMAC-SHA1 → **standard base64 with padding** (`Base64.encodeToString(mac, 2)` = the
NO_WRAP flags, alphabet `+/`, trailing `=`) → 28 characters, the last position is `=`. On the
wire percent-escaping by the native side: `+`→`%2B`, `/`→`%2F`, `=`→`%3D` (UPPERCASE HEX).

LIVE example (Samsung SM-A325F, 2026-09-22): in `/v2/code`
```
token=DZ3O…%3D
```
— 28 chars of b64std (the last `=` → `%3D`), exactly 20 bytes of HMAC (confirmed by the live
capture: "token=20B HMAC b64std with '='"). Between repeated code requests for the same number
the value is the same (HMAC is deterministic, there is no nonce/time in the inputs).

## How it is formed in the app

**The full algorithm — X/I0U.java:20-150:**

**Step 1. secretKey (64 bytes) = PBKDF2WithHmacSHA1And8BIT:**
```java
byteArrayOutputStream.write(AbstractC25022B0c.A1a("UTF-8", packageName)); // "com.whatsapp"
// + ПОЛНОЕ содержимое ресурса about_logo.png (drawable-hdpi / -v4 / xxhdpi-v4, fallback openRawResource)
char[] cArr = <байты потока как char[] — «8-bit» пароль>;
SecretKey secretKeyA08 = C00L.A08("PBKDF2WithHmacSHA1And8BIT", HV3.A00 /*соль*/, cArr, 128, 512);
```
- password (char[]): the UTF-8 bytes of the packageName string (`com.whatsapp`; in the business
  build `com.whatsapp.w4b` — the salt is then ONE AND THE SAME for both builds, only the
  packageName in the password changes), concatenated with the bytes of the PNG resource
  `about_logo.png`;
- the logo source — a strict priority of paths (the first match wins):
  1. `res/drawable-hdpi/about_logo.png`
  2. `res/drawable-hdpi-v4/about_logo.png`
  3. `res/drawable-xxhdpi-v4/about_logo.png`

  In App Bundle releases the drawables moved into density splits (one copy per density), and the
  PNG bytes are the PBKDF2 password: choosing a different density gives a different key and a
  token the server will not accept. The priority "the path before the split, the base APK before
  the split within a path" guarantees the same material for one release; the Java side reads the
  resource via `openRawResource` with the same configuration priority;
- the salt `HV3.A00` (X/HV3.java:7): `Base64.decode("PkTwKSZqUfAUyR0rPQ8hYJ0wNsQQ3dW1+3SCnyTXIfEAxxS75Fw"+
"wNv/c8pP3p0GXKR6OOQmhyERwx74fw1RYSU10I4r1gyBVDbRJ40pidjM41G1I1oN", DEFAULT)` — **90 bytes**
  (the constant is confirmed in classes9.dex @0x2a935e);
- **128 iterations, length 512 bits = 64 bytes** (PBEKeySpec(cArr, salt, 128, 512)).

**Step 2. HMAC-SHA1 over the concatenation of three blocks:**
```java
Mac mac = javax.crypto.Mac.getInstance("HMACSHA1");
mac.init(secretKey);
mac.update(signatures[0].toByteArray());        // 1) certDER
mac.update(md5_of_classes_dex);                 // 2) md5Hash
mac.update(phone.getBytes("UTF-8"));            // 3) phone = in, БЕЗ кода страны
byte[] macBytes = mac.doFinal();                // 20 байт
```
1. **certDER** — `C1Iw.A07(context, packageName)[0].toByteArray()`: the APK signing certificate
   in DER. The source of the material is the **v1 (JAR) signature block** `META-INF/*.RSA|DSA|EC`:
   the certificates are lifted from the PKCS#7 SignedData (the certificates field) in the
   enumeration order; WhatsApp has one certificate, so `[0]` and "all in order" give the same
   thing. An APK without a JAR signature (v2+ only) carries no material for the token. For this
   version (2.26.35.75, base.apk SHA256 fd510aac…): DER 822 bytes, CN=Brian Acton (Android
   Signing Block v2, pair id 0x7109871a); the certificate's SHA256 =
   `OYfQQ9EK769ahxCzZxQY/lfg4ZtlPJ34JVj+tf/OXUQ=`. The Biz build uses the same certificate.
2. **md5Hash** — the MD5 of the **first `classes.dex`**, read from
   `ZipFile(context.getPackageCodePath())` by streaming reads of 8192 bytes (I0U.java:76-123); on
   a read error — the literal `"null"` in UTF-8. For this version:
   `MD5(base.apk/classes.dex) = vRxyuuiy/0ZBAWgWXEEe+w==`.
3. **phone** — the UTF-8 bytes of the national number (`in`, without cc): in the live capture
   `"9206309125"`.

**Step 3. Encoding:** `AbstractC25020B0a.A0p(mac)` = `Base64.encodeToString(mac, 2)` — standard
base64, padding `=`, without line breaks. Into the builder — via `A01` (a plain string); the
percent-encoding (`%2B %2F %3D` UPPERCASE) is performed by the native side.

**Persistence:** none — the token is recomputed on every code request. But the inputs are
deterministic (the key/cert/DEX are fixed per installation, phone is fixed per number), therefore
the value is **the same for all code requests of one number** and is a build constant for all
numbers up to the last block (phone).

## Native implementation

The critical feature of this version: **the resource `about_logo.png` is REMOVED from the APK**
(in resources.arsc the key "about_logo" = noentry in all 16 configurations; the zip has no
`res/drawable*-v4/about_logo.png`; the call is confirmed in the bytecode by the const 0x7f080157
in classes9.dex @0x5f1762). The Java path I0U on this build would fail at `openRawResource`
(Resources$NotFoundException) — which means the actual token in 2.26.35.75 is formed
**natively**: libwhatsapp.so has the import `mbedtls_pkcs5_pbkdf2_hmac`, the rust-crate sha1, and
in rodata — the parameter name "token" and the error "bad_token". A full PBKDF2(128,512)
brute-force over all of res/ (9056 files) and assets/ (279) gave no match for the 64-byte key —
the PBKDF2 input (the logo) is not statically reproducible from this APK; the key could have been
precomputed and embedded/transferred differently. The structure of the HMAC inputs
(certDER || md5Hash || phone) and the encoding at the same time match the live token (20 bytes,
b64std with `=`). The native path, judging by the imports (`mbedtls_pkcs5_pbkdf2_hmac` + sha1),
repeats the same PBKDF2+HMAC-SHA1 derivation scheme — with a precomputed 64-byte key instead of
reading the logo from the APK; the server-side check is then uniform with the classical one (the
`bad_token` error from libwhatsapp rodata — the same server response).

## Value selection conditions (variants, ranges, when absent)

- The raw length is always 20 bytes (HMAC-SHA1); the wire string is always 28 characters of
  b64std with an escaped `=` at the end (`%3D`); among the characters `+`/`/` occur (escaped as
  `%2B`/`%2F`).
- phone = `in` (the national number without cc) — the only variable part.
- Sent in `/v2/code` (and in account-defence requests); in `/v2/register` — ABSENT (live
  capture).
- A platform fork: on **iOS** the client derives the token from a static string and the client
  version — on Android there is no such constant (the material is taken from its own APK), and an
  iOS-form token sent as Android is rejected by the server with `{"reason":"bad_token"}`.
- A re-signed APK (repack): the certificate in META-INF belongs to the repacker → the token is
  formed correctly, but "foreign" — the server answers `bad_token`, indistinguishable from the
  absence of material altogether.
- The Business build (`com.whatsapp.w4b`): the same algorithm and the same salt; the packageName
  in the PBKDF2 password differs (and, accordingly, the key).

## Cryptography and encoding

- PBKDF2-HMAC-SHA1 (the And8BIT variant: the password — bytes as char[], without UTF-16
  conversion), 128 iterations, 512 bits, salt of 90 bytes (HV3.A00).
- HMAC-SHA1 (javax.crypto.Mac "HMACSHA1") with the secretKey key over
  `certDER || md5Hash || phone`.
- Output: Std base64 with padding (flag 2 = NO_WRAP), then percent-encode by the native side
  (UPPERCASE `%XX`); the escaping concerns exactly the three base64 characters — `+`, `/`, `=`
  (`%2B`/`%2F`/`%3D`), for which the URLEncoder and percent semantics are equivalent.
- The constituent constants cross-checked against base.apk 2.26.35.75 (personal): CertKey = DER
  822 B (CN=Brian Acton, SHA256 `OYfQQ9EK769ahxCzZxQY/lfg4ZtlPJ34JVj+tf/OXUQ=`) — a byte-for-byte
  match with the APK Signing Block; HashKey = `vRxyuuiy/0ZBAWgWXEEe+w==` — a match with
  MD5(classes.dex); SecretKey (the 64-byte PBKDF2 product) — statically unverifiable (the
  about_logo.png input was removed from the APK).

## Examples

- Live capture: `token=DZ3O…%3D` — only in `/v2/code`; in register the field is absent.
- Formula: `token = b64std( HMAC-SHA1( PBKDF2_8BIT("com.whatsapp" || about_logo.png, salt90B,
  128 iter, 64B), DER_cert(822B) || MD5(classes.dex)(16B) || "9206309125" ) )`.
- A repeated code request for the same number from the same installation → a byte-for-byte same
  token (all the inputs are unchanged); a number change changes only the HMAC tail; an APK
  reinstall with a different signature changes the certDER block.
