# method, reason — OTP delivery method and the reason for a repeated code request

## Purpose

- **method** — the verification code delivery channel chosen by the user/application: SMS, voice call, flash-call, or autoconf. A pure UI choice, gets into /v2/code.
- **reason** — the reason/context of the current code request: an empty string in the normal flow, non-empty literals — on repeated requests after verification errors and in the autoconf path.

## Place in the flow

| Parameter | /v2/code | /v2/register | /v2/exist |
|---|---|---|---|
| method | yes | **no** | no |
| reason | yes (always, even empty) | no | no |

Both are placed only into the /v2/code map:
- reason — by the collector `IF6.A0H()` (`X/IF6.java:1113`): `put("reason", str)` unconditionally (an empty string also goes on the wire as `reason=`);
- method — by the Kotlin bridge `KotlinRegistrationBridge.A08()` (`com/whatsapp/registration/core/http/KotlinRegistrationBridge.java:1577`): `req.A01("method", str9)` (method A01 — an unconditional put of a string value).

## Format

Strings:

```
method=sms           ← живой пример (SM-A325F: выбран SMS)
reason=              ← живой пример: ПУСТАЯ строка (на проводе reason=)
```

## How it is formed in the application

### method

The value comes from the code request UI branch (VerifyPhoneNumber) as the 4th argument of the service call `A2E(...)` and reaches the bridge:

- `A2E(h8z, str, str2, "sms", HBF.A1T(this), z)` — `VerifyPhoneNumber.java:3839` (code request by SMS — the main path);
- `A2E(h8z, str, str2, "voice", null, z)` — `VerifyPhoneNumber.java:1300` ("Call me");
- `A2E(h8z, str, GCN.A0s(this, str), "flash", null, true)` — `VerifyPhoneNumber.java:704` (flash-call verification);
- `"autoconf"` — the autoconf verification path (AutoconfUseCase; requires a non-empty clientStartMessage, otherwise the request is skipped — `RequestCodeRepository$requestCode$2.java`).

The bridge puts the string into the request and simultaneously remembers it in the pref `registration_last_code_method` (used later to interpret the incoming code). The value is never null — the key is always present on the wire.

### reason

The source — the static field `X.IEz.A00` (`X/IEz.java:35`):

```java
public abstract class IEz {
    public static String A00 = "";      // дефолт — пустая строка
    ...
}
```

`RequestCodeRepository` reads it directly (`String str9 = IEz.A00;` / `String str11 = IEz.A00;` — both paths: standalone verification and the main requestCode) and passes it into `IF6.A0H` as the first string argument → `put("reason", ...)`.

Who writes into `IEz.A00` (the complete list from the reverse engineering):

| Writer | Value | Context |
|---|---|---|
| `RegisterPhone.java:1656` | `""` (reset) | onPause of the number entry screen — zeroing before the next flow |
| `VerifyPhoneNumber.java:2524` | `""` (reset) | start of a new verification cycle |
| `VerifyPhoneNumber.java:1572` | `"format-wrong"` | `A18()` — onFormatWrongError: the server rejected the number format |
| `AutoconfUseCase.java:108` | `AbstractC40441HvU.A00(status)` | the autoconf verification result (see the table below) |
| `RegisterPhone.java:3621` | carrying over the same value | onPause (identity transfer into the metric) |

The complete table of autoconf status values (`X/AbstractC40441HvU.A00`, Integer → string):

| Code | reason value | Meaning |
|---|---|---|
| 1 | VERIFIED_STANDALONE | the number was verified via the standalone path |
| 2 | ERROR_FAIL_TO_INITIALIZE_WAMSYS | wamsys failed to initialize |
| 3 | ERROR_UNSPECIFIED | unspecified error |
| 4 | ERROR_CONNECTIVITY | network error |
| 5 | FAIL_MISMATCH | the code did not match |
| 6 | FAIL_TOO_MANY_GUESSES | too many guess attempts |
| 7 | FAIL_GUESSED_TOO_FAST | the attempts are too fast |
| 8 | FAIL_MISSING | code not found |
| 9 | FAIL_STALE | the code expired |
| 10 | FAIL_TEMPORARILY_UNAVAILABLE | temporarily unavailable |
| 11 | FAIL_BLOCKED | blocking |
| 12 | SECURITY_CODE | security code scenario |
| 13 | ERROR_LIMITED_RELEASE | limited release |
| 14 | DEVICE_CONFIRM_OR_SECOND_OTP | device confirmation or a second OTP is needed |
| 15 | SECOND_OTP | second OTP |
| 16 | FAIL_NOT_ALLOWED | not allowed |
| 17 | FAIL_CONSENT_PENDING | waiting for consent |
| 18 | FAIL_FORMAT_WRONG | wrong format |
| 19 | FAIL_CONSENT_PRIMARY_LINKING_ALREADY_REGISTERED | primary already registered |
| 20 | FAIL_INCORRECT | incorrect code |
| 21 | FAIL_RESET_TOO_SOON | the reset came too soon |
| 22 | FAIL_CHALLENGE_RECAPTCHA | a recaptcha challenge is required |
| 23 | SMS_REQUIRED | SMS required |
| default | YES | autoconf passed successfully |

A separate path of non-empty reasons is resetSecurityCode (`GCL.A0X(...).A0F()`, outside the main /v2/code registration flow).

## Value selection conditions

### method

| Value | Condition |
|---|---|
| **"sms"** | the main path: the "Next" button on the code entry screen / choosing SMS. The dominant case; only with this method does hasav appear in the request |
| "voice" | the user selected "Call me" (voice reading of the code) |
| "flash" | flash-call verification (a missed call); with it hasav = null |
| "autoconf" | autoconf verification via Google/client capabilities (requires a non-empty client_start_message) |

### reason

| Value | Condition |
|---|---|
| **""** (empty) | a normal first code request and all subsequent ones without errors — after the resets the field is always empty by the moment of the request. Live dump: empty |
| "format-wrong" | the previous attempt ended with a number format error → the next /v2/code goes with this reason |
| VERIFIED_STANDALONE / FAIL_* / ERROR_* / SMS_REQUIRED / YES / SECURITY_CODE / SECOND_OTP / DEVICE_CONFIRM_OR_SECOND_OTP | the previous autoconf verification ended with this status → the next code request carries it as reason |

Invariant: reason is the "tail" of the previous attempt; it is non-empty ONLY when the code request follows a verification/autoconf error.

## Examples

- Live dump: `method=sms`, `reason=` (empty) — the first code request, the SMS channel.
- After a number format error: `method=sms`, `reason=format-wrong`.
- After a failed autoconf (the code did not match): `method=sms`, `reason=FAIL_MISMATCH`.
- A retry by voice: `method=voice`, `reason=` (empty; hasav absent).
