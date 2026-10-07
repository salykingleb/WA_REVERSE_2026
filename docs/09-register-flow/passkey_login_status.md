# passkey_login_status — stage of the passkey login attempt (wireToken)

## Position in the flow

A POST /v2/register parameter: the outcome of the last attempt to use a
passkey (WebAuthn login) on the verification screen. Unlike most fields, it
is put into the map not by IF6 methods but directly by the body of the
VerifyCodeRepository$verify$2 coroutine — the persisted stage is read from
the `passkey_login_stage` prefs and converted into the HFx enum's wire token.
It lets the server track the passkey login funnel and decide whether to
offer the passkey again.

## Format (+live example)

A string — one of the five wireTokens (X/EnumC38791HFx.java):

| Enum (ordinal) | value (int, stored in prefs) | wireToken (in the request) | When it occurs |
|---|---|---|---|
| NOT_ATTEMPTED (0) | 0 | **not_attempted** | the passkey was not attempted — the default; any fresh registration |
| NO_CREDENTIALS (1) | 1 | **no_credentials** | PasskeyVerifier received the "no saved credentials" result from the API (passkey_client_login_nopasskey) |
| SUCCESS (2) | 2 | **success** | the passkey login succeeded (passkey_client_login_success) |
| CANCEL (3) | 3 | **cancel** | the user cancelled the system passkey dialog (passkey_client_login_cancelled) |
| ERROR (4) | 4 | **error** | other WebAuthn API errors: ineligible / error codes 0,3,4 (passkey_client_login_ineligible / passkey_client_login_error) |

Live example (dump 2026-09-22): `passkey_login_status=not_attempted`
(a fresh device; the passkey path was never even started).

## How it is formed in the app (class file:line, code)

Reading and putting into the map — the body of the
VerifyCodeRepository$verify$2 coroutine (restored from a jadx instruction
dump):

```java
// VerifyCodeRepository$verify$2.invokeSuspend (dump, L379–L392)
int stage = prefs("passkey_login_stage",
                  EnumC38791HFx.NOT_ATTEMPTED.value);       // default 0
HFx e = Arrays.stream(values()).find{ it.value == stage }
        ?: EnumC38791HFx.NOT_ATTEMPTED;                     // no match → default
map.put("passkey_login_status", e.wireToken.getBytes(UTF8));
```

The enum definition — X/EnumC38791HFx.java (constructor
`EnumC38791HFx(String, int ordinal, int value, String wireToken)`;
wireToken — the exact literals `not_attempted` / `no_credentials` /
`success` / `cancel` / `error`).

Who writes the `passkey_login_stage` prefs — PasskeyVerifier
(com/whatsapp/registration/verification/passkey/PasskeyVerifier.java):

```java
// :153–162 — parsing the PasskeyAndroidApi.A01(...) result (C23089AKv.A02):
//  result code 1 → CANCEL(3):        "passkey_client_login_cancelled"
//  result code 2 → NO_CREDENTIALS(1): "passkey_client_login_nopasskey"
//  codes 0,3,4   → ERROR(4):         ineligible / error
AbstractC49522Fq.A1N(GCN.A0A(prefs), "passkey_login_stage", enumC38791HFx.value);

// :212 — the success branch:
AbstractC49522Fq.A1N(GCN.A0A(prefs), "passkey_login_stage",
                     EnumC38791HFx.A06.value);              // SUCCESS(2)
```

The passkey path is activated only if the server sent a passkey challenge on
/v2/code (or earlier) (see /v2/passkey_auth, IF6.A0k; auth_response in
register). In the normal SMS flow the stage stays 0 → not_attempted.

## Value selection conditions

- **not_attempted** — the default value for any device where the passkey was
  not shown: a fresh registration, SMS/voice/flash without a passkey
  challenge. This is the only value expected in the standard flow.
- The other four — only after the passkey dialog was actually shown in the
  current or a previous session (the prefs survive an app restart):
  - no_credentials — a device/account without a saved passkey;
  - cancel — the user closed the dialog;
  - success — the passkey login succeeded (then usually /v2/passkey_auth
    rather than OTP);
  - error — the API returned an error (no support, ineligible, etc.).
- A value in the prefs not matching any enum is impossible (0–4 cover the
  range), but the code defensively substitutes NOT_ATTEMPTED.

## Cryptography and encoding

- No cryptography: the ASCII wireToken string → UTF-8 bytes → HSJ.A00
  percent-encoding on the wire (underscores are not escaped).
- The field is always present in the request (an unconditional put into the
  map).

## Examples

- Standard SMS registration (live dump): `passkey_login_status=not_attempted`.
- The user cancelled the passkey dialog and then entered the OTP:
  `passkey_login_status=cancel`.
- The passkey login succeeded, but the server required additional OTP
  verification: `passkey_login_status=success`.
