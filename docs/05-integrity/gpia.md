# gpia — encrypted Google Play Integrity APK attestation block (JSON of 8 fields about the APK and the token)

## Place in the flow

The `gpia` parameter is attached to the registration HTTP requests `/v2/code` and `/v2/register`
(as well as to all registration HTTP calls: makeConsentRequest, pre-chatd AB check, verifySecurityCode,
registerPhoneNumber). The formation flow:

1. Gate: `X/IF6.java:594-614 (A0V)` — the parameter is added only when AB flag **3753** is enabled
   (enabled in current builds; when disabled — `Log.w("094F163801F883C27FD4")` and exit).
2. The coroutine `X/J6X.java case 12`: takes the public authkey bytes (`C28040CRt.A04.A0I()`; on null →
   code 1005), does `Base64.encodeToString(raw, 3)` → requestHash, launches the Standard Integrity API
   (`X/C30670DbN.java` — AB timeout 4263, code 1004; `X/C40307Hsz.java` — prepareIntegrityToken).
3. `(token, errorCode)` arrive in `C36467FyF case 2` →
   `JniBridge.jvidispatchIIDOOOO(errorCode, shaRetryDelay, token, application, callback DSY, wajCtx)`
   (file `com/whatsapp/registration/core/integritysignals/F43FA254595FE297CBAE8$fc09ceed2dedd87cc620c$2.java:63-72`).
4. Native code builds the gpia JSON, encrypts it (the shared AES-256-CBC envelope, see
   encryption-key.md) and returns the base64 string via the callback →
   `map.put("gpia", B0W.A1Z(...))`.
5. A0V call sites in the code: `RequestCodeRepository$requestCode$2.java:394` (/v2/code), `IF6.java:1417`
   (checkIfExists), `IF6.java:1566` (generateAuthCode), `IF6.java:1694`.

After registration the server may send the XMPP stanza `<gpia><request nonce=…/></gpia>`
(`X/C27871Nj.java:294-308`) — the response goes natively as `<ib><gpia><jws>…</jws></gpia></ib>`
(`X/C12G.java:31-50`, dispatch id 3); this is a separate jws envelope, not the HTTP parameter gpia.

## Wire format (+ live example)

An ASCII JSON of **strictly 8 keys in the EXACT order** `sizeInBytes, packageName, p, cert, sha256,
shatr, code, token`, compact (no spaces), `/` escaped as `\/`. Then the whole JSON is encrypted
with the envelope (AES-256-CBC, IV||ct, PKCS7, base64.Std) and percent-encoded.

Live plaintext (captured 2026-09-22, WhatsApp 2.26.35.75, /v2/code):

```json
{"sizeInBytes":"127447992","packageName":"com.whatsapp","p":"/data/app/~~MrDQfLH2lFqZVpzZ9xK7Jg==/com.whatsapp-22KXkf5U5ar7mCfZlr8ltA==/base.apk","cert":"OKD31QX+GP7GT780Psqq8xDb15k=","sha256":"/VEKrB9WsYGAMdFIVQPMo396qeQlYvT2bxZtu1gMPgw=","shatr":"p7OAlmVZ1f6oCEU5cPzBS297HP1rkqpcNIPHHe+ukm4=","code":0,"token":"CtMCARCnMGtY4sAcXVJL6…Fwo"}
```

(in the real plaintext all `/` inside values are `\/`; shown without escaping for readability).
`sizeInBytes` is a string (not a number); `code` is a number without omitempty (when 0, `"code":0` is written).

## How it is formed in the application

Fields (all values are computed natively and cached):

| Field | Value source | Live value (2.26.35.75) |
|---|---|---|
| `sizeInBytes` | size of base.apk in bytes, Long→string (getter 0x7cfdd2d3dc) | `"127447992"` |
| `packageName` | package name (getter 0x7cfdd2d2e8) | `com.whatsapp` (for Business: `com.whatsapp.w4b`) |
| `p` | base.apk installation path = ApplicationInfo.sourceDir (JNI getApplicationInfo 0x7cfdd3d318, cache global [0x7cfe391000+0x310]) | `/data/app/~~MrDQfLH2…/com.whatsapp-…/base.apk` |
| `cert` | base64.Std(SHA-1 of the DER signing certificate, 822 B, the first certificate of the APK Signing Block v2/v3; native getter 0x7cfdd2d270) | `OKD31QX+GP7GT780Psqq8xDb15k=` |
| `sha256` | base64.Std(SHA-256 of the WHOLE base.apk), natively `WCAAPISha256Create` | `/VEKrB9WsYGAMdFIVQPMo396qeQlYvT2bxZtu1gMPgw=` |
| `shatr` | base64.Std(SHA-256(base.apk[0:0xA00000])) — the first 10 MiB; WAJIntegrityDataStore cache; on a read error `"-"` | `p7OAlmVZ1f6oCEU5cPzBS297HP1rkqpcNIPHHe+ukm4=` |
| `code` | Play Integrity result code (see below) | `0` |
| `token` | JWS/JWE Play Integrity token; on error the string `"undefined"` (the C string `undefined` in rodata @ro+0x27dd62, the default for a null token) | `CtMC…Fwo` |

For 2.26.37.73 by the same logic: `sizeInBytes="129472282"`,
`sha256="1oUpYgdN0ehe/QGjX23NFK+oj8riGuiJQJIdhGCsJus="`.

The shatr formula is proven byte-for-byte (F_shatr.md): SHA-256 of the first 10 485 760 bytes (0xA00000)
of base.apk; if the APK is smaller than 10 MiB — the whole file; w4b 2.26.36.72 yields
`6Y6oEUYbx2QUCTD1ADlK1GoWUIqHeXaVcY+Ot1cgLHY=`. The value is device-bound — NO, it depends only on
the APK bytes, it can be recomputed without a device dump.

## Native implementation (libwhatsapp.so)

- **`__WCAAPIGenerateGPIAParamOnMainImpl` @0x7cfdd3b9a4…0x7cfdd3c068** (rx+0x5c9a4) — builds
  **three JSONs simultaneously** (registers: x19=gpia, x22=_gg, x21=_gi) with paired key writes.
  The key insertion order by addresses 0x7cfdd3bab4-0x7cfdd3bdb8 (not equal to the wire order — the protobuf-JSON-style
  serializer reorders; the order reference is the live dump):

  | value | gpia key | string address | _gi pair string address |
  |---|---|---|---|
  | token (JWS) | `token` | 0x7cfdd3bab4 | 0x7cfdd3bac4 (`_it`) |
  | errorCode | `code` | 0x7cfdd3bae8 | 0x7cfdd3bafc (`_ic`) |
  | SHA256(base.apk) | `sha256` | 0x7cfdd3bb20 | 0x7cfdd3bb34 (`_is`) |
  | base.apk path | `p` | 0x7cfdd3bb60 | 0x7cfdd3bb74 (`_p`) |
  | SHA1(of the certificate) | `cert` | 0x7cfdd3bbb8 | 0x7cfdd3bbcc (`_icr`) |
  | digest 10 MiB | `shatr` | 0x7cfdd3bd3c | 0x7cfdd3bd78 (`_ist`) |
  | packageName | `packageName` | 0x7cfdd3bd50 | 0x7cfdd3bd90 (`_ip`) |
  | file size | `sizeInBytes` | 0x7cfdd3bd64 | 0x7cfdd3bda8 (`_isb`) |

- The shatr getter: `0x7cfdd3c088(ctx, path, limit)` — the fragment `0x7cfdd3bd0c: mov w2, #0xa00000`
  (the 10 MiB LIMIT), on NULL → the fallback string `"-"` (rodata `b'-\0%s: ig'`); then
  `0x7cfdd2f0b4(pathStr, 0xa00000)` → `0x7cfdd2f0f0(path, limit)`: reading in chunks of
  `min(remaining, 0x100000)` (1 MiB), SHA-256-update of each chunk (0x7cfdd71e44), init 0x7cfdd71df4,
  final = 32 B digest → base64. The function = `WAJIntegrityHashCreateFileSHABase64EncodedData`.
- Cache: the wrapper `0x7cfdd3c5e0` — `WAJIntegrityDataStoreCopyValue` (w1=0); on a miss the key name comes from
  the obfuscated table `0x7cfdd76a64(0x42)`, recomputation, `WAJIntegrityDataStoreSetValue`.
- All 8 keys are found as C strings in runtime-rodata: cert@ro+0x8fcfe, shatr@0xbfd59,
  packageName@0xfdff0, sha256@0x157edc, sizeInBytes@0xb453e.
- Encryption: `_WAJIntegrityCreateEncryptedAES256CBC` @0x7cfdd3b3a4 (see the key file).

## Value selection conditions

- `code = 0` — ONLY with a successfully obtained live Play Integrity token (J6X returns
  `C40616HyW(token, 0)`); simultaneously `token` = JWS.
- `code ≠ 0` + `token = "undefined"` on errors: 1000 unrecognized exception; 1001 no network;
  1002 rate-limit of prepare calls (>5 per 60 s, the counter `pref_gpia_prepare_call_count_in_last_interval`);
  1003 IntegrityTokenProvider not prepared; 1004 timeout (AB 4263, log `on_failure_exception/1004`,
  C30670DbN.java:137/146); 1005 no authkey bytes; negative = `ApiException.getStatusCode()`
  from GMS (−2, −4, −6 in the captures).
- `token` is always written (either JWS, or `"undefined"`); `code` is always written (including 0).
- Freshness between code and register: the parameter is RE-CREATED — the tokens in /v2/code and /v2/register are DIFFERENT
  (a fresh Play Integrity request for each HTTP request), all static fields (sizeInBytes,
  packageName, p, cert, sha256, shatr) are identical; `code=0` in both with a live token.
- The AB 3753 gate: with the flag disabled the parameter is not sent at all.

## Cryptography and encoding

1. JSON (the order `sizeInBytes,packageName,p,cert,sha256,shatr,code,token`, protobuf-JSON style,
   `/`→`\/`).
2. AES-256-CBC, key = SHA256(std-base64(authkey)), IV 16 B random ‖ PKCS7, base64.Std
   (`_WAJIntegrityCreateEncryptedAES256CBC`).
3. Java percent-encoding (X/HSJ.A00): `+`→`%2B`, `/`→`%2F`, `=`→`%3D` UPPERCASE.

The related `<jws>` envelope (XMPP, not HTTP): JSON `{"retries":N,"code":0,"packageName":…,"sha256":…,
"token":JWE,"nonce":echo}` — 3692 b64 ≈ 2768 bytes; the nonce = the server gpia-nonce that the application
puts into `IntegrityTokenRequest.setNonce` (a confirmation of the reverse of `_WCAAPISendNonce`); the token there is a
JWE (`alg A256KW`, `enc A256GCM`, header `eyJhbGciOiJBMjU2S1ci…`).

## Examples

- Success (live, /v2/code 2.26.35.75): `"code":0`, `"token":"CtMCARCnMGtY4sAcXVJL6…Fwo"`;
  in register the token is different, code is also 0.
- Error (observed emulator values): `{"…","code":1004,"token":"undefined"}`,
  or code ∈ {1002, −2, −6, −4}.
- Constants of version 2.26.35.75 (recomputed from base.apk and matched against the capture): sha256
  `/VEKrB9WsYGAMdFIVQPMo396qeQlYvT2bxZtu1gMPgw=`; shatr `p7OAlmVZ1f6oCEU5cPzBS297HP1rkqpcNIPHHe+ukm4=`;
  cert `OKD31QX+GP7GT780Psqq8xDb15k=` (the same for wa and w4b — a shared signing certificate).
