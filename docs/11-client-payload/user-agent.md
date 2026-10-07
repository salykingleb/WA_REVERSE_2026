# Field 5 userAgent (X/C1Ht) — complete field map

`userAgent` is the nested protobuf message `X/C1Ht.java` (original `X.1Ht`), field
No. 5 of ClientPayload (`USER_AGENT_FIELD_NUMBER = 5`, `X/C1HS.java:54`). It is
populated entirely by the filler `X.1Hs` (DI case 6556, `X/C1US.java:19805-20011`) —
the same code works both for first login and for resume; the value changes only when
the build/locale/SIM changes.

## Fields of the C1Ht message

The numbers are constants from `X/C1Ht.java:15-32`:

| # | Field | Type | Source in the filler | Condition |
|---|---|---|---|---|
| 1 | `platform` | enum `C1Hy` | hardcoded `C1Hy.ANDROID` (0) | always |
| 2 | `appVersion` | message `C1I0` | `C1Hz.A00()` | always |
| 3 | `mcc` | string | `C1Id.A00(TelephonyManager.getNetworkOperator()).A00` | only if TelephonyManager != null |
| 4 | `mnc` | string | `C1Id.A00(...).A01` | same |
| 5 | `osVersion` | string | `C1Ie.A05` = `Build.VERSION.RELEASE` | always |
| 6 | `manufacturer` | string | `C1Ie.A03` = `Build.MANUFACTURER` | always |
| 7 | `device` | string | `C1Ie.A00` = `Build.DEVICE` | always |
| 8 | `osBuildNumber` | string | `C1Ie.A02` = `Build.DISPLAY` | always |
| 9 | `phoneId` | string | `C1Ih.Aso().A01` (installation fdid) | always |
| 10 | `releaseChannel` | enum `C1Ir` | — | not set by this filler |
| 11 | `localeLanguageIso6391` | string | `C0FK.A0A()` | if non-empty and != `zz` |
| 12 | `localeCountryIso31661Alpha2` | string | `C0FK.A09()` | if != `ZZ` |
| 13 | `deviceBoard` | string | `C1Ie.A01` = `Build.BOARD` | only if the string is non-empty |
| 14 | `deviceExpId` | string | `StringUtils.A09(prefs.A0J().A03())` — lowercase | always (the value may be empty) |
| 15 | `deviceType` | enum `C1In` | based on `C1Il.A00()` | always |
| 16 | `deviceModelType` | string | `C1Ie.A04` = `Build.MODEL` | always |
| 17 | `distributionChannel` | enum `C1Iu` | — | not set by this filler |

The filler first takes the existing `userAgent_` from the builder (in practice —
`C1Ht.DEFAULT_INSTANCE`) and extends it (`X/C1US.java:19824-19828`); at the end it
writes the finished message to `c1hs.userAgent_` with bit 4 of `bitField0_`
(`X/C1US.java:19965-19970`).

## platform (1)

Unconditionally `C1Hy.ANDROID` = 0 (`X/C1US.java:19829-19835`). The full enum
`X/C1Hy.java` for reference: ANDROID 0, IOS 1, WINDOWS_PHONE 2, BLACKBERRY 3, ...,
SMB_ANDROID 10, KAIOS 11, WINDOWS 13, WEB 14, GREEN_ANDROID 16, WEAROS 29,
ARDEVICE 30, VRDEVICE 31 — only 0 is ever reached on this client.

## appVersion (2)

`C1Hz.A00()` (`X/C1Hz.java:8-31`) — parsing the string `"2.26.35.75"` baked into the
build, with the separator `"\\."` and a limit of 4 parts. Fewer than three parts or
a non-numeric part — `AssertionError` ("expected at least three parts in version
name"). The result is the array `[2, 26, 35, 75]`; the filler spreads it into the
message `C1I0` (`X/C1I0.java:16-20`):

- `primary` (1) = 2, `secondary` (2) = 26, `tertiary` (3) = 35 — always the first three;
- `quaternary` (4) = 75 — only if the array has length 4 (`X/C1US.java:19858-19864`);
- field 5 `quinary` exists in the schema but is not set by this build.

## mcc / mnc (3, 4)

`X/C1US.java:19871-19886`:

```java
TelephonyManager tm = ((C0AO) ...).A0K();
if (tm != null) {
    C1Id c1IdA00 = C1Id.A00(tm.getNetworkOperator());
    c1Ht4.mcc_ = c1IdA00.A00;
    c1Ht5.mnc_ = c1IdA01.A01;
}
```

The parser `X/C1Id.java`: the `getNetworkOperator()` string is run through the regex
`(\d{3})(\d{2,3})` (full match). On a match — mcc = the first 3 digits, mnc = the
remainder, padded via `String.format(Locale.US, "%03d", ...)` (a two-digit MNC "01"
becomes "001"). On no match or a null string — the pair `("000", "000")` is returned,
and **the fields are still set with the value 000/000**.

Consequence: the fields are absent from the payload only when `TelephonyManager` is
unavailable at all (`A0K()` returned null). "No SIM" by itself does not remove the
fields: without a SIM, `getNetworkOperator()` returns an empty string, the regex does
not match, and `mcc="000"`, `mnc="000"` go on the wire — exactly this is seen on the
live phone from the live capture: "Samsung SM-A325F, Wi-Fi WITHOUT SIM: mcc=000/mnc=000".

## osVersion / manufacturer / device / osBuildNumber / deviceBoard / deviceModelType

All come from the singleton `X/C1Ie.java:15-34` — plain reads of `android.os.Build`
at creation:

| C1Ht field | C1Ie field | Build constant | Live value (SM-A325F) |
|---|---|---|---|
| `osVersion` (5) | `A05` | `Build.VERSION.RELEASE` | Android version (11/12/13) |
| `manufacturer` (6) | `A03` | `Build.MANUFACTURER` | `samsung` |
| `device` (7) | `A00` | `Build.DEVICE` | platform codename |
| `osBuildNumber` (8) | `A02` | `Build.DISPLAY` | `TP1A.220624.014.A325FXXSBDXJ1` (live capture) |
| `deviceBoard` (13) | `A01` | `Build.BOARD` | only if `length() != 0` (`X/C1US.java:19908-19914`) |
| `deviceModelType` (16) | `A04` | `Build.MODEL` | `SM-A325F` |

## phoneId (9) — installation fdid

`X/C1US.java:19920-19925`: `((C1Ih) this.A05.A00.get()).Aso().A01`.

`X/C1Ih.java:12-26` (`Aso()`): the fdid lives in the prefs `phoneid_id` (string) +
`phoneid_timestamp` (long). If either is missing — a `UUID.randomUUID()` is
generated, saved, and returned. `A01` of `C1Ij` is the UUID string itself. The live
value of this installation (from the `/v2/code` capture): fdid =
`da1e6667-b5ed-4fa6-9828-53283393c119` (UUID v4). The same fdid goes as the `fdid`
parameter in HTTP registration — userAgent and `/v2/register` must be consistent.

## deviceExpId (14) — experiment id

`X/C1US.java:19926-19931`:

```java
String strA09 = StringUtils.A09(((C018208l) this.A0A.A00.get()).A0J().A03());
c1Ht13.deviceExpId_ = strA09;   // нижний регистр
```

The value is the installation's expid, read from the prefs via `C018208l.A0J()` and
lowercased with `StringUtils.A09` (locale-independent lowercase). The live value from
the HTTP capture: `expid=AbCmAYhROweMJWInj2dh_A` — it will go into the payload as
`abcmayjhrowemjwinj2dh_a`. This is a UUID v3 (Block Store, live capture), not
re-randomized on every login — stable for the installation.

## deviceType (15)

`X/C1US.java:19932-19949` — a switch on the ordinal of the `C1Il.A00()` result:

| ordinal `EnumC26891Im` | Classification | `C1In` value on the wire |
|---|---|---|
| 1 (TABLET) | tablet | `TABLET` = 1 |
| 2 (VR) | VR headset | `VR` = 4 |
| 3 (DESKTOP) | desktop mode | `DESKTOP` = 2 |
| everything else (MOBILE, FOLDABLE, AMBIGUOUS, WEARABLE...) | phone | `PHONE` = 0 |

`X/C1Il.java:43-63` (`A01()`): the device type is cached in the pref
`pref_device_type`; if the pref is empty — it is computed by the heavyweight `A00()`
(screen/fold analysis; the body did not decompile in normal jadx mode) and saved.
The full classification enum `EnumC26891Im`: Mobile, Tablet, Vr, Desktop, Foldable,
Ambiguous, Wearable, Wearable_WhatsApi — but the protobuf enum `C1In` has five
values: PHONE 0, TABLET 1, DESKTOP 2, WEARABLE 3, VR 4 (`X/C1In.java:8-14`).
On SM-A325F — `PHONE` (0).

## locale (11, 12)

`X/C1US.java:19950-19964`, the source is `C0FK` (the application locale):

- `localeLanguageIso6391` (11) = `C0FK.A0A()` — ISO 639-1 language; set only if
  non-empty and not `zz`;
- `localeCountryIso31661Alpha2` (12) = `C0FK.A09()` — ISO 3166-1-alpha-2 country;
  set only if not `ZZ` (an empty country is allowed and will go as an empty string).

Live capture values (live capture: `lg=ru`, `lc=RU`):
`localeLanguageIso6391 = "ru"`, `localeCountryIso31661Alpha2 = "RU"`.

## Final view for the live installation (SM-A325F, Wi-Fi without SIM)

```text
platform                = ANDROID (0)
appVersion              = {primary:2, secondary:26, tertiary:35, quaternary:75}
mcc                     = "000"      # SIM нет, но TelephonyManager есть
mnc                     = "000"
osVersion               = Build.VERSION.RELEASE
manufacturer            = "samsung"
device                  = Build.DEVICE
osBuildNumber           = "TP1A.220624.014.A325FXXSBDXJ1"
phoneId                 = "da1e6667-b5ed-4fa6-9828-53283393c119"
localeLanguageIso6391   = "ru"
localeCountryIso31661Alpha2 = "RU"
deviceBoard             = Build.BOARD (если непустой)
deviceExpId             = "abcmayjhrowemjwinj2dh_a"
deviceType              = PHONE (0)
deviceModelType         = "SM-A325F"
```

## Where to look

- Filler: `X/C1US.java:19805-20011` (case 6556, X.1Hs).
- Schema: `X/C1Ht.java`, `X/C1I0.java`, enums `X/C1Hy.java`, `X/C1In.java`, `X/C1Ir.java`.
- Version: `X/C1Hz.java`; Build fields: `X/C1Ie.java`; fdid: `X/C1Ih.java`, `X/C1Ij.java`.
- mcc/mnc: `X/C1Id.java` (regex and `%03d` format).
- Device type: `X/C1Il.java` + `X/EnumC26891Im.java`.
