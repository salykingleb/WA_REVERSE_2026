# cc / in — country code and national phone number

## Purpose

`cc` (country code) and `in` (input number / internal national number) — the identifier of
the phone number being registered, the only parameters in the group whose value is set by
the user manually. `cc` is the telephone country code without the `+` sign (digits only),
`in` is the national significant number without the country code and without the national
trunk prefix (typically without a leading `0`). Together, `cc + in` form the full number in
"E.164-without-plus" form, under which the server creates the account, and the app — the
local storage keys (see the section about the `id` storage key).

Present in `/v2/code` and `/v2/register` (as well as in `/exist`, `/v2/consent` and
client-funnel). The value does not change between requests of the same session.

## Place in the flow

1. The user selects a country in the spinner and enters the number on the `RegisterPhone`
   screen (Countries + Enter Phone Number screen).
2. `RegisterPhone` normalizes the input (digits only, trunk prefix stripping) and saves the
   cc/in pair to SharedPreferences.
3. `RequestCodeRepository` / `VerifyCodeRepository` pass cc/in to
   `KotlinRegistrationBridge`, where they are put into the request `A0R(...)` → `cc`, `in`.
4. The storage key `id` (`IF6.A0u`) is also derived from cc/in BEFORE sending `/v2/code`.

Wire order (the app's native order): the parameters do NOT go side by side —
`in` comes earlier, `cc` later:
- `/v2/code`: `..., token, in, expid, simnum, cc, ...`
- `/v2/register`: `..., entrypoint, in, expid, simnum, cc, ...`

## Wire format (+live example)

Both are plain strings, ASCII digits only, without `+`, spaces, parentheses, or hyphens.
They are put via `C40787I4g.A01(name, str)` (a simple string, no extra encoding); during
serialization they end up in the percent-encoded query via `HSJ.A00` — %XX hex digits in
UPPERCASE (for digits the encoding is an identity, but this is part of the common
transport).

Live capture 2026-09-22 (Samsung SM-A325F, Wi-Fi without SIM):

```
cc=7&in=9206309125
```

(in the full query: `...&token=...&in=9206309125&expid=...&simnum=0&cc=7&...` — in both
requests, code and register, the values match byte for byte).

## How it is formed in the app

### cc

- The source is the user's choice: the country spinner of the number entry screen. The
  selected country is a metadata object `C1IS` (a row from `res/raw/countries`, a TSV with
  12+ columns; parser `X/C14540lG.java:23-67`), field `A00` is the int country code. In the
  result object `X/C1IF.java:18` this is `countryCode_` (int).
- The country pre-selection in the UI may use `TelephonyManager.getSimCountryIso()`
  (`RegisterPhone.java:2455`) — but this is ONLY a spinner hint, it does not get into the
  request.
- If the user types the number with a `+`, the app parses it and determines cc by prefix;
  the entered string is cleaned in any case: `replaceAll("\\D", "")`
  (`RegisterPhone.java:1208`) — plus signs, spaces, parentheses, hyphens are removed. The
  result is parsed into an int: `Integer.parseInt(...)` (`RegisterPhone.java:1224`).
- It is put into the request as a string: `KotlinRegistrationBridge.A0R` —
  `com/whatsapp/registration/core/http/KotlinRegistrationBridge.java:1066-1069`:
  ```java
  public static void A0R(C40787I4g c40787I4g, String str, String str2) {
      c40787I4g.A01("cc", str);    // cc — plain string
      c40787I4g.A01("in", str2);
  }
  ```
  The `cc`/`in` keys are also duplicated in the funnel request (`A02("cc", str6)` /
  `A02("in", str7)`, KotlinRegistrationBridge.java:103-104).

### in

- User input: national number. Cleaning is the same — `replaceAll("\\D", "")`
  (`RegisterPhone.java:1225`).
- Then the trunk prefix is removed: `C14540lG.A02(int cc, String number)`
  (`X/C14540lG.java:102-210`; the call — `RegisterPhone.java:1227` via the `A0V` wrapper;
  the same algorithm in `CountryAndPhoneNumberFragment`):
  - using the `res/raw/countries` metadata for the selected cc, leading characters of the
    number that match the NDC prefixes (`C1IS.A0A`) are enumerated and stripped;
  - special countries cc `7, 241, 998, 992` go through a separate branch with length checks
    against the number length array `C1IS.A05` (min/max) — for them the trunk strip is
    effectively not performed (C14540lG.java:116);
  - typical effect: the leading `0` is discarded (GB `07911…` → `in=7911…`).
- Example of the full chain: `+7 (912) 345-67-89` → cc field `7`, in field
  `912) 345-67-89` → `replaceAll("\\D","")` → `9123456789` → the trunk strip for cc=7 does
  not change it → `in=9123456789`.

### The role of cc/in in the `id` storage key

The `cc + in` combination is not only wire parameters but also the local storage key for the
`id` value (a request parameter):

- `X/IF6.java:1246-1256` (`IF6.A0u(cc, in)`):
  ```java
  String strA00 = AbstractC39008HQk.A00(AbstractC49602Fy.A0V(str, str2)); // ключ = маска(cc+in)
  byte[] bArrA0I = C00L.A0I(application, strA00);   // чтение из files/rc2
  if (bArrA0I != null) return bArrA0I;
  byte[] bArrA0G = C00L.A0G();                      // 160-бит AES-ключ (KeyGenerator)
  C00L.A09(application, strA00, bArrA0G);           // запись в files/rc2
  return bArrA0G;
  ```
  where `AbstractC49602Fy.A0V(cc, in)` is a simple concatenation of `cc + in` without `+`
  (X/AbstractC49602Fy.java:188-192), and `AbstractC39008HQk.A00`
  (X/AbstractC39008HQk.java:8-13) compresses the number with the regex
  `^([17]|2[07]|3[0123469]|4[013456789]|5[12345678]|6[0123456]|8[1246]|9[0123458]|\d{3})\d*?(\d{4,6})$`
  down to "cc + last 4–6 digits" (middle masking).
- The storage is the encrypted file `files/rc2` (`X/C00L.java:206-237`): AES/OFB writes,
  the key is derived by a PBK function from the string `C0UA.A0X + <mask(cc+in)>` — that is,
  IDENTITY MEMORY IS TIED TO THE NUMBER.
- The `A0u` result is exactly the wire parameter `id`: `KotlinRegistrationBridge.A0T`
  (KotlinRegistrationBridge.java:1078-1084) puts `c40787I4g.A05("id", bArr)` — that is, the
  `id` of every request is the 160-bit key read/generated under the storage keyed by cc+in
  (calls: RequestCodeRepository$requestCode$2.java:342/387, IF6.java:683/794/884/1307/1672).
- The cc/in pair is additionally cached in the `RegisterPhone` SharedPreferences:
  `com.whatsapp.registration.RegisterPhone.input_country_code` / `input_phone_number`
  (RegisterPhone.java:886, 3423) and `country_code` / `phone_number`
  (RegisterPhone.java:1252-1264).

## Value selection conditions (all variants)

| Situation | cc | in |
|---|---|---|
| User selected a country and entered a number | int code of the selected country from `res/raw/countries` | number digits after `replaceAll("\\D","")` and the trunk strip |
| User typed the entire number with `+` | country code parsed by prefix | the rest of the number without cc and the trunk prefix |
| Input with spaces/parentheses/hyphens | equivalent to clean input — garbage is removed by the regex | the same |
| Number with a leading `0` (national trunk) | unchanged | the leading `0` (or another NDC prefix) is stripped by `C14540lG.A02`; for cc 7/241/998/992 — NOT stripped |
| Wi-Fi device without SIM | cc/in do NOT depend on SIM — the only source is user input and the spinner (the SIM hint `getSimCountryIso` only affects pre-selection) | likewise |
| Roaming / second SIM | unchanged — cc/in do not read TelephonyManager on sending | unchanged |

The app has no "cc must match mcc" constraint: the combination of cc(number) + lg/lc(device
locale) is free — the "number of one country, UI of another" combination is fully legal
(see lg-lc.md).

## Examples

- Live dump (RU number, Wi-Fi without SIM): `cc=7`, `in=9206309125` — identical in /v2/code
  and /v2/register.
- User input `+7 (912) 345-67-89` → `cc=7`, `in=9123456789`.
- User input `07911 123456`, country GB → `replaceAll("\\D","")` = `07911123456`, trunk
  strip of the leading `0` → `cc=44`, `in=7911123456`.
- Input with a plus in the cc field is impossible in a real request: `replaceAll("\\D", "")`
  cuts out the `+` before the parameters are assembled.
