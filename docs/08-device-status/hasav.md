# hasav — SMS automatic verification availability (Google Play Services SMS Retriever)

## Purpose

The parameter tells the server which mechanism the application will be able to use to automatically catch the incoming SMS with the verification code:

- **"2"** — Google Play Services **SMS Retriever** is active (automatic verification without permissions);
- **"1"** — the retriever is not working, but the runtime permission RECEIVE_SMS has been granted (the classic SMS reception via `SMS_RECEIVED`);
- **"0"** — neither of the two: the code will have to be entered manually.

This is NOT "auto-fill" (autofilling the input field) — it is precisely the choice of the automatic OTP interception channel. The server uses the value to decide whether to offer flash/autoconf mechanics and how to interpret the client's subsequent behavior.

## Place in the flow

| Request | Presence |
|---|---|
| /v2/code | **yes — but ONLY with method=sms** |
| /v2/register | no |
| /v2/exist | no |

Placed by the /v2/code map collector `IF6.A0H()` (`X/IF6.java:1114–1116`): the value (the string `autoVerification`) is placed only if != null. With method="voice" and method="flash" the application passes null — the parameter is ABSENT on the wire. In /v2/register the field does not exist at all.

Live dump of 2026-09-22: hasav=2 is present in /v2/code (method=sms) and absent in /v2/register.

## Format

The string "0" | "1" | "2" (UTF-8 bytes):

```
hasav=2        ← живой пример (Samsung SM-A325F, GMS/Play установлен, method=sms)
```

## How it is formed in the application

The computation — `HBF.A1T(VerifyPhoneNumber)` (`X/HBF.java:26–32`):

```java
public static String A1T(VerifyPhoneNumber v) {
    if (v.A1i) return "2";                                      // SMS Retriever активен
    return v.A0m.A02("android.permission.RECEIVE_SMS") == 0     // checkSelfPermission
        ? "1" : "0";
}
```

- `A0m.A02(...)` — a wrapper of `Context.checkSelfPermission()` (`X/C06740Ti.java:91`).
- The flag `A1i` has **priority over the permission**: if the retriever is active, the value "2" is set even with RECEIVE_SMS granted.

Route into the request: VerifyPhoneNumber → `A2E(h8z, str, str2, "sms", HBF.A1T(this), z)` (`com/whatsapp/registration/app/verifyphone/VerifyPhoneNumber.java:3839`) → RequestCodeUseCase ($autoVerification) → `IF6.A0H` → the key `hasav`.

### Where the A1i flag (the retriever) comes from

1. Back on the number entry screen (RegisterPhone), pressing "Next" calls `SmsRetrieverUtils.maybeUseSmsRetriever` (`X/AbstractC40442HvV.A00`, call from `RegisterPhone.java:554`):
   - Google Play Services availability check: `GooglePlayServicesUtil.checkGooglePlayServicesAvailable(ctx, 12451000)` (version 12451000 is historical, any modern GMS passes it);
   - then `SmsRetriever.API.startSmsRetriever()` — requires NO permissions;
   - success → the pref `registration_use_sms_retriever=true` (`X/C42548Itz:140`) + a callback into RegisterPhone → VerifyPhoneNumber starts with the extra `use_sms_retriever=true` → in onCreate (`VerifyPhoneNumber.java:4058–4060`) the flag **A1i=true**;
   - failure (no GMS / the task crashed) → the "proceedWithoutSmsRetriever" branch → only then is the runtime permission RECEIVE_SMS requested (dialog requestCode 701, `VerifyPhoneNumber.java:7534–7541`): granted → A1i per AB flag 21677; denied → A1i=false.
2. Immediately before the code request, "maybeUseSmsRetriever" is executed once more (`VerifyPhoneNumber.java:1653–1662`): if A1i is already set — the request goes immediately; otherwise another attempt to start the retriever, and only on failure — dialog 701.

## Value selection conditions

| Value | Condition | When it actually occurs |
|---|---|---|
| **"2"** | A1i=true — the GMS SMS Retriever is started (the permission does NOT matter) | Any device with working GMS/Play: the retriever starts even before the code entry screen, without a single dialog. The dominant case (~85–97% of real registrations) |
| **"1"** | A1i=false AND RECEIVE_SMS granted | GMS missing/broken (Huawei etc.) and the user granted the permission in the dialog; or a reinstall over a previously granted permission |
| **"0"** | A1i=false AND no permission | No GMS + denial in the dialog, or the dialog was not shown |

Parameter presence:

| Situation | hasav on the wire |
|---|---|
| /v2/code, method=sms | always present ("0"/"1"/"2") |
| /v2/code, method=voice (`VerifyPhoneNumber.java:1300` — A2E(..., "voice", null, ...)) | absent |
| /v2/code, method=flash (`VerifyPhoneNumber.java:704` — A2E(..., "flash", null, ...)) | absent |
| /v2/register | always absent |

## Examples

- Live dump: SM-A325F (GMS/Play present, root, Wi-Fi without SIM), method=sms → `hasav=2`.
- A phone with Play Store and a SIM with the SMS permission granted → still `hasav=2` (A1i is checked first).
- Huawei without GMS, the user granted the SMS permission → `hasav=1`.
