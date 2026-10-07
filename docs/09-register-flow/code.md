# code — the 6-digit OTP code for number confirmation

## Position in the flow

The main POST /v2/register parameter: a one-time code received by the user
via /v2/code (SMS / voice call / flash-call / e-mail) and sent back to
complete registration. It gets into the request directly from the `$code`
argument of the VerifyCodeRepository$verify$2 coroutine → the builder's `code`
field (line A01). This name is exactly `code`; `security_code` is a parameter
of a different flow (the 2FA password, see extra-params.md); the OTP is never
sent under it.

## Format (+live example)

- A string of digits, usually **6 characters** (`A5N` only allows sending a
  digit string of length `A2H` (=6): `if (str == null || str.length() != A2H) return;`).
- No separators, no prefixes, no encoding on top — "as the user entered it /
  as it came in the SMS".

Live example (dump 2026-09-22): `code=111111` (test OTP).

## How it is formed in the app (class file:line, code)

Sources of the code (four paths, all converging in VerifyPhoneNumber.A5L):

1. **Manual SMS code entry** — the CodeInputField → `A5M(String)`:
   VerifyPhoneNumber.java:1359–1376, saves the code to prefs
   (`com.whatsapp.registration.VerifyPhoneNumber.sms_code`), then
   `A5L(h8a, str, cc, number, "sms", 2)` (:1375).
2. **Manual voice/wa_old/email_otp entry** — `A5N(String)`:
   VerifyPhoneNumber.java:1378–1434; determines the method string
   (`"email_otp"` / `"wa_old"` / `"voice"`, :1408–1416) and calls
   `A5L(h8a, str, cc, number, strA01, 1)` (:1421). Before that, A5N checks
   that the length == 6 and all characters are digits.
3. **Flash-call (automatic number detection from the call)** — the `ByC`
   callback: VerifyPhoneNumber.java:1437–1446 →
   `A5L(h8a, str, cc, number, "flash", 2)`.
4. **Autofill of the saved SMS code** (SMS retriever / Activity state
   restoration): `onCreate` reads the saved code
   (`icq.A05(cc, number)` — prefs sms_code/sms_cc/sms_phone_number) and, if
   present, sends `A5L(h8a, strA05, cc, number, "sms", 4)` —
   VerifyPhoneNumber.java:4287 (context: :4270–4290, condition
   `strA05 != null && !GCO.A1S(state) && mode != 6`).

Then the chain (in detail in call-map.md):

```java
// VerifyPhoneNumber.java:1342-1356
public void A5L(H8a h8a, String str /*code*/, String str2 /*cc*/,
                String str3 /*in*/, String str4 /*method*/, int i /*entered*/) {
    ...
    AbstractC49512Fp.A1X(new VerifyCodeUseCase$verifyCode$1(
        null /*bzp*/, h8d, h8a, str, str4, str2, str3,
        A12(this) /*authCodeContext*/, null, null, i, this.A01), ...);
}
```

```java
// VerifyCodeRepository$verify$2 (invokeSuspend dump): $code passes as r29/r43
//   into registerPhoneNumberBlocking → bridge A07
// KotlinRegistrationBridge.java:1301
c40787I4gA01.A01("code", str8);   // str8 = OTP, A01 encoding — the string as is
```

The second A01 argument passes the check `C000800h.A0A(str2, 1)` (non-null,
non-empty) — an empty code physically cannot get into the request.

## Value selection conditions

- `code` is always sent (it is the only mandatory "secret" register parameter
  besides the e2e bundle).
- The value depends on the delivery channel from /v2/code: a 6-digit SMS
  code, 6 digits from the end of the flash-call number, an e-mail OTP (also
  6 digits), a voice code.
- Resending the same code does not change the format; the attempt counter
  goes into `client_metrics.attempts` (in the live register dump: attempts=0 —
  the code was entered on the first attempt after the last requestCode).
- The separate `method` parameter is **not sent** in /v2/register (null-guard
  A02 + r31 gating in verify$2: the $method value reaches the bridge only in
  alternative flows or when codeVerificationMode==6). The delivery channel is
  reflected for the server in `entered`, not in `method`.

## Cryptography and encoding

- No encryption: the code is sent in plaintext inside the wamsys TLS session.
  Protection comes from the e2e bundle itself (authkey/e_*), id/backup_token,
  and the session registration token at the transport level.
- Encoding: `A01("code", str)` — the string without transformations (ASCII
  digits); percent-encoding on the wire does not change digits.
- Contrast: the 2FA password with the "password" method is encrypted (J3Z →
  CanonicalPasswordService) and put into `code` only in the 2FA flow via
  verifySecurityCodeViaRegister — that is a different call (IF6.A0n), not the
  OTP path.

## Examples

- SMS, manual entry: `code=482915` (+ `entered=2`).
- Voice, manual entry: `code=482915` (+ `entered=1`).
- Autofill from a saved SMS: `code=482915` (+ `entered=4`).
- Live dump: `code=111111`, `entered=1` (voice/wa_old path, mode=0).
