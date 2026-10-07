# advertising_id — Google Advertising ID (GAID), an optional parameter

## Place in the flow

`advertising_id` — the device's Google Advertising ID (GAID). The only identifier of the group
that can be **completely absent** on the wire: the value is computed anew before every request
via `IF6.A0p(cc, entrypoint)`, and if the function returned null — the key is not put
(`C40787I4g.A02` skips null). It participates in `/v2/code`, `/v2/register`, `/v2/consent`,
account-defence and funnel-log. Call points: code → `if7.A0p(str2=$cc, "code_entrypoint")`
(RequestCodeRepository$requestCode$2.java:414); register → `IF6.A06` → `A0p(str,
"security_entrypoint")` (IF6.java:313); consent → `A0p(..., "consent_entrypoint")`. The way onto
the wire: KotlinRegistrationBridge.java:236/351/1226/1307/1587/1945 —
`c40787I4g.A02("advertising_id", id)`.

## Wire format (+LIVE example from the capture)

A plain GAID string: a UUID format of 36 characters, lowercase, with hyphens (usually v4-like
`xxxxxxxx-xxxx-4xxx-yxxx-xxxxxxxxxxxx`), WITHOUT transformations (the A02 method puts it as is;
UUID characters require no percent-encoding).

LIVE EXAMPLE — ABSENCE: in the capture of 2026-09-22 (Samsung SM-A325F, an RU number cc=7, Wi-Fi
without a SIM) the advertising_id parameter is **ABSENT BOTH IN `/v2/code` AND IN
`/v2/register`**. The reason is proven on the device: limit ad tracking is enabled
(`adid_settings.xml` of GMS: `enable_limit_ad_tracking=true`,
`enable_limit_ad_tracking_reason=2`). The zero UUID
`00000000-0000-0000-0000-000000000000` is absent too — the new branch is active (see below),
which nulls the field under LAT.

## How it is formed in the app

**`IF6.A0p(String cc, String entry)` (X/IF6.java:951-965):**
```java
public String A0p(String str, String str2) {
    if (AbstractC31531as.A0f((C05D)this.A05, HWG.A00))               // AB-флаг 33973 ВКЛ
        return ((C39837Hky) C05D.A03(this.A0B)).A00(C02S.A00, str); // НОВАЯ ветка (GoogleAdIdManager)
    try {
        c39744HjQA00 = !"eu".equals(((C14540lG) C05D.A03(this.A08)).A03(str)) ? I2T.A00(this.A03) : null;
    } catch (GU5 | C38815HHb | IOException e) {
        Log.e("RegistrationHttpManager/RegistrationHelper/getAdvertisingId at " + str2 + " failed", e);
    }
    return (c39744HjQA00 != null) ? c39744HjQA00.A00 : null;
}
```
Important details:
- The cc argument is the **telephone country code of the NUMBER** (not MCC; in the live capture
  "7"), therefore Wi-Fi without a SIM does not hinder region determination.
- `tosRegion(cc)` = `X/C14540lG.A03`: parses cc as an int, looks it up in `res/raw/countries`
  (`base.apk:res/j42.tsv`, 243 lines, ≥14 columns; **the region is column 13**, column 2 is the
  telephone code). Region `"eu"` → null. An IOException on read → "treating as EU" → null.

**The new branch `X/C39837Hky.A00(mode=0, cc)` (GoogleAdIdManager):**
1. `pref_pre_chatd_ab_next_fetch_time > 0` AND AB flag 20346 OFF → **null** (the pre-chatd AB
   gate);
2. the cc region (per countries.tsv) = `"eu"` → **null** (there are 48 EU codes in total: 262,30,
   31,32,33,34, 351,352,353,354,356,357,358,359,36,370,371,372,376,377,378,379,385,386,39,40,41,
   420,421, 423,43,44,45,46,47,48,49,590,594,596); RU (cc=7) → region `"row"`;
3. otherwise `I2T.A00(context)` — the embedding of `AdvertisingIdClient.getAdvertisingIdInfo`
   (X/I2T.java):
   - no package `com.android.vending` (Play Store) → `HHb(9)` → **null**;
   - `GoogleApiAvailability` (minVersion 12451000) unavailable (code != 0/2) → IOException →
     null;
   - the bind to `com.google.android.gms.ads.identifier.service.START` failed / a 10 s timeout
     (CAPTURE_OPERATION_TIMEOUT_MS) / Interrupted / RemoteException → IOException → null;
   - success: transact 1 = getId, transact 2 = isLimitAdTrackingEnabled(true) →
     `C39744HjQ{id, limitTracking}`: **limit == true → null**; otherwise id (a null string from
     GMS — also null).

**The old branch** (AB flag 33973 OFF): without the pre-chatd gate; with limit=true it would
still have returned id (under LAT GMS returns `00000000-0000-0000-0000-000000000000` — the field
WOULD HAVE BEEN on the wire).

**Persistence:** none — `A0p` is invoked anew on every request, nothing is cached. But the
device's GAID itself is stable (it changes only by a reset in the Google settings), therefore
within a code/register/consent session the value is the same. In the live capture it was not sent
anywhere.

## Native implementation

The value is Java; it is put into the builder via `A02` (a plain string, NOT included in the
"already encoded" Set). The name `"advertising_id"` is present in the native rodata table of
registration parameters (libwhatsapp.so @0x2f35e0+). When null, the key is simply absent from the
map — the native side does not see it.

## Value selection conditions (variants, ranges, when absent)

The parameter is sent ONLY when ALL the conditions hold:
1) the telephone code of the NUMBER is not in the EU region (the 48 codes above; the region comes
   from the number's cc, SIM/MCC are unimportant);
2) the pre-chatd AB gate is not active;
3) Play Store (`com.android.vending`) is installed and a working GMS ≥ 12451000;
4) the bind to the AdvertisingId service succeeded (a 10 s limit);
5) limit ad tracking is OFF (the default of "healthy" Play devices);
6) GMS returned a non-empty id.
→ then `advertising_id` = GAID (36 chars, a lowercase UUID, the same in code/register/consent).

The field is absent: EU numbers (always), devices without Play Services (Chinese/Huawei
profiles), devices with LAT enabled (the live RU reference SM-A325F), any AdvertisingIdClient
errors/timeouts, the pre-chatd gate. Indirect confirmation from the live capture: in the WhatsApp
logs (files/Logs, 2026-09-21/22) there is not a single "getAdvertisingId … failed" line — there
were no exceptions, I2T routinely got limit=true and returned null.

## Cryptography and encoding

No cryptography. A plain UUID string via `A02`; percent-encoding by the native side is a no-op
for UUID characters.

## Examples

- Live capture: the parameter is ABSENT in code and register (LAT enabled; indirectly — the new
  branch is active: there is not even a zero-UUID).
- An example of presence (a typical Play device without LAT, a non-EU number):
  `advertising_id=cbea42b1-3cd4-4e6f-9a1b-2f3d4e5f6a7b` — the same in all the session's requests.
- The consent paths mirror the condition: `X/I7I:165,194` — `if (str != null && str.length() > 0)`.
