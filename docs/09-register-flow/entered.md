# entered — the numeric code of the OTP entry method

## Position in the flow

A POST /v2/register parameter telling the server HOW exactly the user entered
the OTP code: manually from the keyboard, autofilled from SMS, or via the
voice/e-mail path. It is put into the additional core map of IF6.A0I and
encoded as a plain string. It is linked to `code` (see code.md for the value)
and indirectly reflects the delivery channel, since `method` is not sent in
register.

## Format (+live example)

A digit string: `"1"`, `"2"`, or `"4"` (int → String.valueOf). No other values
occur in A5L calls.

Live example (dump 2026-09-22): `entered=1` (the voice/wa_old/email_otp path).

## How it is formed in the app (class file:line, code)

```java
// X/IF6.java:377 (inside A0I — the core of the verify additional parameters)
linkedHashMapA0O.put("entered",
    AbstractC87053t4.A1b(String.valueOf(i), C07k.A05));   // i = codeEntryMethod
```

The value of `i` (codeEntryMethod) is set by the entry point — the last
argument of VerifyPhoneNumber.A5L (signature: `A5L(H8a, code, cc, in, method,
int i)`, VerifyPhoneNumber.java:1342). All call sites:

| Value | Call site | Scenario | File:line |
|---|---|---|---|
| **2** | `A5M(String)` | manual SMS code entry from the keyboard (`"sms", 2`) | VerifyPhoneNumber.java:1375 |
| **2** | `ByC(String,String)` | flash-call: the code = the last digits of the caller's number (`"flash", 2`) | VerifyPhoneNumber.java:1446 |
| **1** | `A5N(String)` | voice code / e-mail OTP / wa_old path (`strA01, 1`; strA01 ∈ {"voice","email_otp","wa_old"}) | VerifyPhoneNumber.java:1421 |
| **4** | `onCreate` (restore) | autofill of a previously saved SMS code (`"sms", 4`), condition: a saved code exists, the state is not "already entered", and mode != 6 | VerifyPhoneNumber.java:4287 |

The value then passes unchanged: A5L → VerifyCodeUseCase$verifyCode$1
(field `$codeEntryMethod`) → VerifyCodeRepository$verify$2 (field
`$codeEntryMethod`) → IF6.A0I(..., i).

## Value selection conditions

- **2** — "entered by a human from the keyboard" (SMS or flash). The main
  path of the live app during SMS registration.
- **1** — a voice call, e-mail OTP, or "wa_old" (re-verification of an
  existing number). This is exactly the value in the live dump of 2026-09-22.
- **4** — the code is not typed by the user: it is pulled from the saved
  sms_code (SMS retriever / screen restoration) and sent automatically
  in onCreate.
- The value is determined exclusively by the entry method on the specific
  screen; codeVerificationMode (normal 0 / dynamic 2FA 6) does not affect it
  (with mode=6 the autofill at :4287 is blocked, the other paths are the
  same).
- The autoconf path (C38457Gyi, carrier instant verification) also sends
  entered=codeEntryMethod, but that is a separate /v2/autoconf* flow, not
  /v2/register.

## Cryptography and encoding

- No cryptography: `String.valueOf(int)` → UTF-8 bytes
  (AbstractC87053t4.A1b) → HSJ.A00 percent-encoding on the wire (digits do
  not change).
- The field is always present (an unconditional put in A0I).

## Examples

- Manual SMS entry: `code=482915&entered=2`.
- Voice code: `code=482915&entered=1` (live dump: `code=111111&entered=1`).
- SMS autofill: `code=482915&entered=4`.
