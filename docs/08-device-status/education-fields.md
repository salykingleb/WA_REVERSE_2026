# education_screen_displayed, clicked_education_link, prefer_sms_over_flash — flash-call education state

## Purpose

A trio of parameters of the flash-call verification education screen (PrimaryFlashCallEducationScreen):

- **education_screen_displayed** — whether the flash-call education screen was shown to the user;
- **prefer_sms_over_flash** — whether the user chose SMS instead of a flash call in the education;
- **clicked_education_link** — whether the user clicked the link in the education (the "What is this?" link); the field is ABSENT on the wire if there was no interaction.

All three describe the same UI history of the device, and in a fresh installation they are in the default state (false / false / absence).

## Place in the flow

| Parameter | /v2/code | /v2/register | /v2/exist |
|---|---|---|---|
| education_screen_displayed | yes (always, even "false") | no | no |
| prefer_sms_over_flash | yes (always) | no | no |
| clicked_education_link | conditional (only after an interaction) | no | no |

The first two are placed by the collector `IF6.A0H()` (the /v2/code map) unconditionally. The third is NOT in the IF6 map: it is added by the Kotlin bridge `A08()` and only with hfr=PHONE (not the 2FA path).

## Format

The strings "true"/"false" (or absence):

```
education_screen_displayed=false
prefer_sms_over_flash=false
(Clicked_education_link отсутствует)      ← живой пример (SM-A325F, свежая установка)
```

## How it is formed in the application

### education_screen_displayed — `X/IF6.java:1119`

```java
map.put("education_screen_displayed", String.valueOf(
    GCN.A0D(A02(if6)).getBoolean("pref_flash_call_education_screen_displayed", false)));
```

- The pref `pref_flash_call_education_screen_displayed`, **default false**;
- set to true after PrimaryFlashCallEducationScreen is actually shown;
- **reset to false at the start of each registration activity** (`X/AbstractActivityC38711HBf.java:128`: `A1U(prefs, "pref_flash_call_education_screen_displayed", false)`) — that is, the value lives only within the current verification pass.

### prefer_sms_over_flash — `X/IF6.java:1120`

```java
map.put("prefer_sms_over_flash", String.valueOf(
    GCN.A0D(A02(if6)).getBoolean("pref_prefer_sms_over_flash", false)));
```

- The pref `pref_prefer_sms_over_flash`, **default false**;
- set to true ONLY in the flash-call education screen with Eligible status and the SMS choice: `X/C42022Ikh.java:59–60` —
  `A1U(prefs, "pref_primary_flash_call_status", "primary_eligible"); A1U(prefs, "pref_prefer_sms_over_flash", true);`
- reset to false at the start of a registration activity (`X/AbstractActivityC38711HBf.java:129`).

### clicked_education_link — the bridge `KotlinRegistrationBridge.A08()` (`:1579–1581`)

```java
c40787I4gA01.A00("clicked_education_link", i);   // только при hfr != HFR.A03 (не-2LA/PHONE-путь)
```

where `i = GCN.A0D(prefs).getInt("pref_flash_call_education_link_clicked", -1)` — read in `VerifyPhoneNumber.A2E` (`VerifyPhoneNumber.java:1288`, the local `i3`).

The key mechanics of `C40787I4g.A00(name, int)` (`X/C40787I4g.java:53–64`) — mapping an int to a string with "swallowing":

```java
public final void A00(String str, int i) {
    if (i == 0)      map.put(str, "false");
    else if (i == 1) map.put(str, "true");
    /* любое другое значение (в т.ч. -1) → ключ НЕ добавляется вообще */
}
```

## Value selection conditions

### education_screen_displayed

| Value | Condition |
|---|---|
| **"false"** | default: the education screen was not shown in the current pass (a fresh installation — always) |
| "true" | PrimaryFlashCallEducationScreen was shown in the current verification pass (the pref was set and not yet reset by the activity start) |

### prefer_sms_over_flash

| Value | Condition |
|---|---|
| **"false"** | default: education not completed / the user did not choose SMS |
| "true" | in the flash-call education the user chose SMS delivery (the pref was set in C42022Ikh and not reset) |

### clicked_education_link

| State | Pref value | On the wire |
|---|---|---|
| no interaction (default) | -1 | **the key is ABSENT** (the standard case of a fresh installation — live dump) |
| saw the education, did not click the link | 0 | `clicked_education_link=false` |
| clicked the education link | 1 | `clicked_education_link=true` |

Correlation: clicked_education_link=0/1 on the wire is possible only together with education_screen_displayed=true (the screen must have been shown for it to be clicked). The combination displayed=false + clicked=true is a forgery marker.

## Examples

- Live dump (SM-A325F, fresh installation, SMS flow without education): `education_screen_displayed=false`, `prefer_sms_over_flash=false`, no clicked_education_link key.
- A device with the education shown and SMS chosen: `education_screen_displayed=true`, `prefer_sms_over_flash=true`, possibly `clicked_education_link=true`.
