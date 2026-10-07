# _gg — encrypted Google Play Integrity result block (JSON {_ic, _it}: code + token)

## Place in the flow

The `_gg` parameter (native name from the obfuscated table, sid 84) is the "condensed result" of the Play Integrity
request: the error code and the token itself. It is attached to all registration HTTP requests
(`/v2/code`, `/v2/register` and the other registration calls) via the shared native package:

```
X/IF6.java:553-567 (A0S):
    JniBridge.jvidispatchIOO(7, app, waj)                         // native state preparation
    map2 = JniBridge.jvidispatchOOO(16, app, waj)                 // the underscore-parameter map
    map.putAll(map2)                                              // _gg among them
```

The value (a base64 string of the encrypted JSON) is formed natively by the same builder as gpia/_gi
(`__WCAAPIGenerateGPIAParamOnMainImpl`, the register x22 = _gg), from the (errorCode, token) pair obtained
from the Java Play Integrity pipeline. The wire position in the request: in /v2/code _gg is 7th (after
sim_mnc/id/mnc), in /v2/register — 8th.

## Wire format (+ live example)

Strictly two keys, the order `_ic, _it` (written as a pair next to gpia's token/code, string addresses
0x7cfdd3bae8-0x7cfdd3bb04):

```json
{"_ic":0,"_it":"CtMCARCnMGtY4sAcXVJL6…Fwo"}
```

- `_ic` — a number (int), always written, including 0.
- `_it` — a string: the Play Integrity JWS/JWE token, or the literal `"undefined"` on error.

Live example /v2/code (2.26.35.75): `{"_ic":0,"_it":"CtMCARCnMGtY4sAcXVJL6…Fwo"}` (JWS).
Live example 2.26.37.73 (the login jws envelope, the same token family):

```
eyJhbGciOiJBMjU2S1ciLCJlbmMiOiJBMjU2R0NNIn0.cRBoBfP5….IGM5reMakLSxk7x1CnkYfQ
```

— a real **Play Integrity JWE**: the protected header `{"alg":"A256KW","enc":"A256GCM"}`,
then encrypted key ‖ IV ‖ ciphertext ‖ tag (JWE-compact dot separators).

Then the JSON is encrypted with the shared envelope (AES-256-CBC, key SHA256(std-base64(authkey)), IV||ct, PKCS7,
base64.Std) and percent-encoded by the Java side (X/HSJ.A00).

## How it is formed in the application

The Java pipeline for obtaining (errorCode, token):

1. `X/IF6.java:594 (A0V)` — the AB flag 3753 gate.
2. `X/J6X.java case 12` (862-903): the public authkey bytes (`C28040CRt.A04.A0I()`; null →
   code 1005) → `Base64.encodeToString(raw32B, 3)` — the STANDARD base64 with flag 3 =
   NO_PADDING|NO_WRAP (without `=`), this is the **Play Integrity requestHash (nonce)**; then the token request
   via `C30670DbN` (AB timeout 4263 → code 1004) and `C40307Hsz`.
3. `X/C40307Hsz.java` — **StandardIntegrityManager** (`IntegrityManagerFactory.createStandard`),
   `PrepareIntegrityTokenRequest.builder().setCloudProjectNumber(293955441834L)` (line 65),
   `prepareIntegrityToken(...)`; then `StandardIntegrityTokenRequest` with requestHash →
   `StandardIntegrityTokenProvider.request(...)` → the token.
4. The result `(token, errorCode)` → `C36467FyF case 2` →
   `JniBridge.jvidispatchIIDOOOO(errorCode, shaRetryDelay=0.0, token, application, callback DSY, wajCtx)`
   → native builds the _gg JSON, encrypts it, returns the string.

The native preparation path (live confirmation 2.26.37.73, Frida log):
`jvidispatchIIIIDOOO args=["0","62949436","855397460"]` — preparation of **cloud project number
62949436** and the constant long 855397460. Per jadx (F43FA…$2.java:66-69, X/DTA.java:143) the full
call: `jvidispatchIIIIDOOO(i3, 62949436L, 855397460L, 796.6509679599703d, application, wajCtx,
byte[20] zeros)`, where `i3 = AB(12965) ? 19 : 0` (in the live login AB 12965 is disabled → selector 0).
This is a separate native preparation channel (cloud project 62949436) in addition to the Java channel
prepareIntegrityToken with cloud project 293955441834.

## Native implementation (libwhatsapp.so, base 0x7cfd633000)

- The builder `__WCAAPIGenerateGPIAParamOnMainImpl` @0x7cfdd3b9a4…0x7cfdd3c068 — x22=_gg is written
  pairwise with gpia: `token`(gpia)@0x7cfdd3bab4 / `_it`(_gg)@…ac4; `code`@0x7cfdd3bae8 / `_ic`@0x7cfdd3bafc.
  I.e. `_gg._it` and `gpia.token` are one value; `_gg._ic` and `gpia.code` are one value.
- Keys in the rodata dump: `_ic`@ro+0x140aa9, `_it`@ro+0x999c4; the parameter name `_gg` is sid 84
  of the obfuscated table 0x7cfdd76a64.
- Encryption: `_WAJIntegrityCreateEncryptedAES256CBC` @0x7cfdd3b3a4 (IV 16 B ‖ PKCS7 ‖ base64.Std),
  the key via `WAJIntegrityCreateEncryptedStringUsingKey` (call @0x7cfdd3b828-0x7cfdd3b8a0).
- Related reverse-engineered symbols: `_WCAAPIIntegrityGetStatus`, `_WAJIntegrityTriggerGPIAWithRequestAttributes`,
  `_WCAAPISendNonce` (the echo of the server nonce into setNonce for the XMPP jws).

## Value selection conditions

The exact list of `_ic` codes (Java, `X/J6X.java case 12` + `X/C40307Hsz.java`):

| Code | Condition |
|---|---|
| `0` | success — a live token received (JWS/JWE); simultaneously `_it` = the token |
| `1000` | unrecognized exception (the default catch) |
| `1001` | no network (`C38825HHn(1001)`, log "_NONETWORK", C40307Hsz.A00) |
| `1002` | rate-limit of prepare calls: >5 per 60 s (`pref_gpia_prepare_call_count_in_last_interval >= 5` with `pref_last_gpia_prepare_call_timestamp` inside the 60000 ms window; log "_TOOMANY") |
| `1003` | IntegrityTokenProvider not prepared (NULL) |
| `1004` | request timeout (`C233311z`; the timeout = AB 4263; log `on_failure_exception/1004`, C30670DbN.java:137/146) |
| `1005` | authkey bytes unavailable (`A0I()==null`, J6X case 12) |
| `<0` | `ApiException.getStatusCode()` from GMS/Play (−2, −4, −6 observed) |

With any `code ≠ 0` the token is null → `_it = "undefined"` (the C string `undefined` in rodata @ro+0x27dd62 —
the default value for a null token).

**Freshness**: the token is requested anew for every HTTP request — in /v2/code and /v2/register the
`_it` tokens are **DIFFERENT** (a FRESH parameter), while within a single request `_gg._it` == `gpia.token`
(one value, inserted twice), `_ic=0` in both with a live token.

The nonce convention (important for reproduction): requestHash = `Base64.encodeToString(authkey_pub_raw
32B, 3)` — std-base64 WITHOUT padding/wrap of exactly the **raw public authkey** (the client's static pair,
`X/C1Jp.java`, `X/AnonymousClass139.java:491 A0I`), and NOT SHA256(authkey). The server may
compare the requestHash from the decrypted token with the registration authkey.

## Cryptography and encoding

1. JSON `{"_ic":N,"_it":"…"}` (compact; `/` in the token is escaped as `\/` before encryption —
   base64url tokens contain no slashes, and neither does the JWE-compact form).
2. The AES-256-CBC envelope: key = SHA256(standard base64(authkey)), IV 16 random bytes ‖ PKCS7,
   base64.Std.
3. Java percent-encoding (`+`→`%2B`, `/`→`%2F`, `=`→`%3D` UPPERCASE).

The related XMPP envelope `<jws>` (after login, the response to `<gpia><request nonce='…'>`):
`{"retries":2,"code":0,"packageName":"com.whatsapp","sha256":"…","token":JWE,"nonce":<echo of the server
nonce>}` — the token is of the same nature (A256KW/A256GCM), the nonce is put into `IntegrityTokenRequest.setNonce`.

## Examples

- Success: `{"_ic":0,"_it":"eyJhbGciOiJBMjU2S1ciLCJlbmMiOiJBMjU2R0NNIn0.cRBoBfP5….IGM5reMakLSxk7x1CnkYfQ"}`
  (JWE length ~2768 bytes; the envelope 3692 b64).
- Error: `{"_ic":1002,"_it":"undefined"}` (or 1003/1004/1005/−2/−4/−6).
- Observed in live captures: `_ic=0` in code and register, `_it` different (a fresh token for every request).
