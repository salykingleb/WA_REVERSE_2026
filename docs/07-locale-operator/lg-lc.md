# lg / lc — language and country of the app UI locale

## Purpose

`lg` (language) and `lc` (locale country) — the language and country of the **app UI
locale** (the device's system locale or the language selected by the user inside WhatsApp).
This is NOT the SIM locale, NOT the telephone country prefix of the number, and NOT
geolocation: the source is exclusively the app's Resources configuration. The server sees
in them "what language the device runs in".

Present in ALL registration requests: `/v2/code`, `/v2/register`, `/v2/exist`, `/v2/consent`,
funnel. Between code and register of the same session the values are identical (computed
from the same process locale).

## Place in the flow

Assembly: `IF6.A0E/A0F` (`X/IF6.java:1094-1100`) → `C0FK.A0A()` (lg) / `C0FK.A09()` (lc) —
put into the request by the builder `KotlinRegistrationBridge.A0S`
(`com/whatsapp/registration/core/http/KotlinRegistrationBridge.java:1071-1076`) together
with `fdid`/`expid`.

Wire order:
- `/v2/code`: `lg` after `network_radio_type` — `..., network_radio_type, lg, rc, pid, ...`;
  **`lc` — the LAST parameter of the request** (50th position).
- `/v2/register`: `..., lg, rc, network_radio_type, pid, ...`; `lc` — again last
  (49th position).

## Wire format (+live example)

- `lg` — 2–3 lowercase ASCII letters: `ru`, `en`, `de`, `pt`, `zh`, `fil`…; the fallback for
  an invalid value is the literal `zz`.
- `lc` — 2 UPPERCASE letters (`RU`, `US`, `GB`, `IN`…) or 3 digits (a UN code of the
  `Numeric` kind): the regex allows `[0-9]{3}`; the fallback is the literal `ZZ`.

Live capture 2026-09-22 (Samsung SM-A325F, system locale ru-RU, Wi-Fi without SIM):

```
lg=ru&lc=RU
```

in /v2/code and /v2/register — the same (paired with cc=7; the match of the locale country
and the number's country is a special case here, not the rule).

## How it is formed in the app

The chain (class `X/C0FK.java` — WhatsAppLocale):

1. The active locale — `C0FK.A0S()` (lines 297-305) → `A03(Configuration)`
   (lines 196-208):
   ```java
   locale = Build.VERSION.SDK_INT >= 24
       ? (configuration.getLocales().isEmpty() ? Locale.getDefault()
                                               : configuration.getLocales().get(0))
       : configuration.locale;
   // если null → Locale.getDefault(); если и он null → Locale.US (крайний, недостижимый фолбэк)
   ```
2. `lg = C0FK.A0A()` (272-283, cache in A0D) → private `A01()` (348-360):
   `A0S().getLanguage()` with validation `C0NE.A02 = Pattern.compile("[a-z]{2,3}")`
   (X/C0NE.java:17). No match → log `WhatsAppLocale/getLanguageInternal/invalid-language`
   and return of **`"zz"`** (lines 351-359).
3. `lc = C0FK.A09()` (367-379): `A0S().getCountry()` with validation
   `C0NE.A03 = Pattern.compile("[A-Z]{2}|[0-9]{3}")` (X/C0NE.java:16). No match →
   **`"ZZ"`**.
4. The language selected inside the app ("Settings → App language", pref `forced_language`)
   is applied by the method `C0FK.A0U(str)` (315-346): saving the tag to prefs,
   `this.A04 = Locale.forLanguageTag(str)` and — the key point — `Locale.setDefault(...)`,
   after which BOTH `Configuration` AND all formatters operate from this selected locale. If
   the user has not changed the language (`forced_language` is empty) — the device's system
   locale is active.
5. Sending: `KotlinRegistrationBridge.A0S` — `c40787I4g.A01("lg", str)` /
   `c40787I4g.A01("lc", str2)` — plain string; then the common percent-encoding (`HSJ.A00`,
   %XX in UPPERCASE — an identity for a-z/A-Z).

Encoding: plain ASCII strings; Cyrillic/diacritics in lg/lc themselves are impossible (the
regexes allow ASCII only).

## Value selection conditions (all variants)

| Situation | lg / lc | Note |
|---|---|---|
| System language = the country's language (typical) | device locale, e.g. `ru`/`RU` | live dump: lg=ru, lc=RU with cc=7 |
| User changed the language INSIDE WhatsApp (`forced_language`) | the selected locale, not the system one | `C0FK.A0U` + `Locale.setDefault`; e.g. system en-US + selected Russian → lg=ru, lc=RU |
| Expatriate/tourist: number of country A, device in language B | cc=A, lg/lc from the device locale (e.g. cc=7, lg=en, lc=US) | a fully legal combination — cc/in and lg/lc are independent |
| Second SIM / roaming | no effect: lg/lc do NOT read TelephonyManager | mcc/mnc can be "foreign" with unchanged lg/lc |
| Language without a country (lg present, lc empty) | lg valid, lc → `ZZ` | `getCountry()` == "" does not match `[A-Z]{2}|[0-9]{3}` |
| Invalid/corrupted language | `zz` | validation fallback |
| Extreme fallback (Locale.getDefault()==null) | `en`/`US` (Locale.US) | unreachable on a real device |
| Language ≠ country of the same locale (en-IN, pt-BR, zh-SG) | taken as is: `en`/`IN`, `pt`/`BR` | the language↔country pairing is not enforced |

Relation to neighboring parameters: the decimal separator of `device_ram`
(`DecimalFormat("#.##")` in `IF6.A0Q` — X/IF6.java:511-521) is taken from the same
effective locale (`ru` → comma `3,56`, `en` → period `3.56`; live dump: `device_ram=3,56`
with lg=ru). That is, the triple lg+lc+device_ram separator are projections of ONE locale
and are always mutually consistent; combinations like "lg=ru/lc=RU + device_ram=3.56" are
never produced by the real app.

## Examples

- Live dump: `lg=ru&lc=RU` (code and register), `device_ram=3,56` — a consistent Russian
  locale.
- A device from the USA, number +1: `lg=en&lc=US`, `device_ram=3.56`.
- A British expatriate with a Russian number: `cc=7`, `in=...`, `lg=en&lc=GB` — legal.
- A Brazilian device: `lg=pt&lc=BR` (not es/BR and not en/BR).
- Indian English: `lg=en&lc=IN` — the language and country of one locale, but different
  "language countries".
- The user selected "Deutsch" in WhatsApp settings on an en-US phone: `lg=de&lc=DE`
  (forced_language overrides both the configuration and Locale.getDefault).
