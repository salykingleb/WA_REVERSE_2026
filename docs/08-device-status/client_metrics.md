# client_metrics — request JSON metrics (attempts, install source, SIM indicators)

## Purpose

A nested JSON object with the request's behavioral metrics: the code request attempt number, the application install source (Install Referrer), and — in specific paths — indicators of the number matching the SIM. Serialized with `JSONObject.toString()` (compact JSON, key insertion order) and sent as an ordinary string parameter.

## Place in the flow

| Request | Presence | Builder class |
|---|---|---|
| /v2/code | yes | `H8Z` (put in `IF6.A0H`, `X/IF6.java:1117`) |
| /v2/register | yes | `H8a` (put in `IF6.A0I`, `X/IF6.java:376`) |
| /v2/exist | yes (+ the additional field was_activated_from_stub) | `H8Z` (`IF6.A0j:1385–1387`) |

Class hierarchy:

```
C39796HkI (база)   → attempts:int, app_campaign_download_source:str? (только если != null)
 ├─ H8Z  (/v2/code, /v2/exist)  → + is_sim_number:int?, is_sim_absent:bool?,
 │                                is_permission_granted:bool?, isUserChoosingToMigrate…:bool?
 └─ H8a  (/v2/register)         → + flash_call_end_success:bool?, no_flash_call_id_received:bool?,
                                  invalid_flash_call_received:bool?   (поля is_sim_number НЕТ ВООБЩЕ)
```

## Format (+live example)

Compact JSON (no spaces), keys in insertion order: attempts → app_campaign_download_source → the optional ones.

Live dump of 2026-09-22 (SM-A325F, SMS path):

```
/v2/code:     client_metrics={"attempts":7,"app_campaign_download_source":"google-play|unknown"}
/v2/register: client_metrics={"attempts":0,"app_campaign_download_source":"google-play|unknown"}
```

Both WITHOUT is_sim_number — in the application's SMS path this field is not filled in (see below).

## How it is formed in the application

### attempts

**/v2/code — the code request counter, growing with each request** (`X/IEz.java:436–441`, `A0E`):

```java
public static H8Z A0E(C018208l c018208l) {
    C1DE c1de = c018208l.A0W();
    int i = c1de.A02().getInt("reg_attempts_generate_code", 0) + 1;   // прочитать + 1
    AbstractC203528ts.A1D(c1de, "reg_attempts_generate_code", i);     // персист
    return new H8Z(i, c018208l.A0M().A04());
}
```

Called from all code request points: SMS (`VerifyPhoneNumber.java:3834`), voice (`:6117`), flash (`:686`), `SendSmsUseCase.java:71/191`, `AccountTransferManager.java:95`, `RegisterPhone.java:961`. That is, attempts = the ordinal number of the code request on the device: the first → 1, each "Resend" → 2, 3, … The live 7 = the seventh code request on the device.

**/v2/register — read WITHOUT an increment from the pref, which is never written in this version → always 0**:

- `VerifyPhoneNumber.java:4040` (onCreate): `this.A13 = H8a.A00(c018208l, c018208l.A07())`;
- `C018208l.java:6815–6817`: `A07() = prefs.getInt("reg_attempts_verify_code", 0)`;
- across the whole jadx tree there is NO writer for `reg_attempts_verify_code` (only reg_attempts_generate_code / reg_attempts_check_exist / reg_attempts_verify_2fa / reg_attempts_device_confirmation are written). The live dump confirms: register attempts=0.

### app_campaign_download_source

The value = `C1C9.A04()` (`X/C1C9.java:15–21`): the pref `app_install_source` (default `"unknown|unknown"`), or `app_install_source_from_app_manager` under referrer_clicked_time conditions. Written by the InstallReferrer handler `X/RunnableC42366Iqh.java:315–330`: it parses the Play referral query and forms the string **`utm_source + "|" + utm_campaign`**; an empty referrer → `"unknown|unknown"`.

An organic installation from the Play Store: the referral `utm_source=google-play&utm_medium=organic`, utm_campaign absent → **`"google-play|unknown"`** — exactly the live value. Persisted on the device, identical in code and register (both factories pass the same `A0M().A04()`).

### is_sim_number (only H8Z, and even then not always)

Set ONLY in the flash-call path — `VerifyPhoneNumber.java:686–687` (maybeAttemptFlashCall): `h8z.A03 = i`, where i is the result of comparing the entered number with the list of SIM numbers:

| i | Condition |
|---|---|
| 1 | the number matched one of the SIM numbers |
| 0 | the SIM number list is non-empty, but there is no match |
| -1 | the list is EMPTY (no SIM / no phone permission) |

In the main code request paths — SMS (`VerifyPhoneNumber.java:3833–3840`) and voice (`:6117–6123`) — A03 is not filled in → null → **the field is absent from the JSON** (live dump of the SMS path: no field). In H8a (register) the is_sim_number field does not physically exist.

### The remaining optional fields

- `is_sim_absent` (bool) = `getSimState()==1` (SIM_STATE_ABSENT) — set only in the voice path (`:6119`);
- `is_permission_granted` (bool), `isUserChoosingToMigrateFromConsumerAppDirectly` (bool) — the flash/biz migration paths;
- `was_activated_from_stub` (bool, pref downloader_stub) — EMBEDDED into the JSON only in /v2/exist (`IF6.A0j:1385–1387`).

For a regular SMS registration none of them is sent.

## Value selection conditions (summary table)

| Field | Always? | Values and conditions |
|---|---|---|
| attempts | yes | code: >= 1, the code request number (1 = the first, grows on resend); register: always 0 (a dead pref in 2.26.35.75) |
| app_campaign_download_source | yes (if the referrer was obtained) | `"google-play\|unknown"` — Play organic (the dominant case); `"unknown\|unknown"` — the referral was not delivered; other `utm_source\|utm_campaign` — campaigns |
| is_sim_number | no | absent in the SMS/voice paths and always present in register; 1/0/-1 only in the flash-call branch |
| is_sim_absent | no | only the voice path |
| is_permission_granted, isUserChoosing… | no | flash/biz migration |
| was_activated_from_stub | no | only /v2/exist |

## Examples

- Live dump: code `{"attempts":7,"app_campaign_download_source":"google-play|unknown"}`; register `{"attempts":0,"app_campaign_download_source":"google-play|unknown"}`.
- The first code request on a new device: code `{"attempts":1,"app_campaign_download_source":"google-play|unknown"}`.
- The flash-call path: code `{"attempts":1,"app_campaign_download_source":"google-play|unknown","is_sim_number":1,...}`.
