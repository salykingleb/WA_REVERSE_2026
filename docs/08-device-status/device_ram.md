# device_ram — device RAM size in gibibytes

## Purpose

An environment status parameter: reports the device's total RAM in gibibytes (GiB) to the server. Used by the server as part of the device profile (anti-fraud/registration environment): it marks the device class (1–2 GiB — budget, 4–8 — flagship, etc.). The value is determined by the device's hardware and locale — it is not a random number.

## Place in the flow

| Request | Presence |
|---|---|
| /v2/code | yes |
| /v2/register | yes |
| /v2/exist (checkIfExists) | yes |

Placed by the shared collector `IF6.A0Q()`, which is called from the maps of all three requests. Cached in the field `IF6.A01` for the process lifetime — across sequential requests of one session the value is guaranteed to be identical.

## Format

A number-string with 0–2 digits after the decimal separator, **the separator depends on the device's system locale**:

```
device_ram=3,56        ← живой пример (Samsung SM-A325F, локаль ru)
device_ram=3.56        ← то же устройство с en-локалью
device_ram=4           ← целое число ГиБ: дробная часть отбрасывается целиком
device_ram=3,5         ← один знак: хвостовой ноль отброшен ("3,50" невозможен)
```

Live capture of 2026-09-22 (SM-A325F): `device_ram=3,56` — with a COMMA, because the device's system locale is Russian.

## How it is formed in the application

The source — `X/IF6.java:511–521`, method `A0Q()`:

```java
str = new DecimalFormat("#.##").format(
        AbstractC40471qk.A02(AbstractC49572Fv.A0g(if6.A0P)) / 1.073741824E9d);
if6.A01 = str;                       // кэш на процесс
map.put("device_ram", B0W.A1Z(str)); // UTF-8 bytes
```

The chain for obtaining the number:

1. `AbstractC40471qk.A02()` (`X/AbstractC40471qk.java:90–99`): creates `new ActivityManager.MemoryInfo()`, fills it via `ActivityManager.getMemoryInfo(memoryInfo)` and returns **`memoryInfo.totalMem`** — the device's total memory in BYTES (Android API: `ActivityManager.MemoryInfo.totalMem`, the field is available since API 16).
2. Division by the constant `1.073741824E9` = 2^30 = 1073741824 — converting bytes to gibibytes. For example, totalMem = 3 823 104 000 bytes (3.5625 GiB on the SM-A325F with ~3.5 GB) → 3.5625.
3. `new DecimalFormat("#.##")` formats:
   - the pattern `#.##` — a maximum of 2 digits after the separator;
   - **trailing zeros are dropped** by DecimalFormat: 4.0 → "4", 3.50 → "3,5", 3.5625 → rounded to 2 digits → "3,56";
   - **the decimal separator is taken from the JVM/system default locale** (`DecimalFormat` without an explicit `DecimalFormatSymbols` uses `Locale.getDefault()`): ru/de/fr → comma, en_US → period.

Why the live capture has a comma: the phone was running with the Russian system locale (in the same capture `lg=ru`, `lc=RU`), so DecimalFormat substituted `,`. This is NOT a protocol quirk — on a device with an en_US locale the same phone would send `3.56`.

## Value selection conditions

The value is fully determined by two factors:

| Factor | Effect |
|---|---|
| The device's `ActivityManager.MemoryInfo.totalMem` | the numeric magnitude: typical values 1–12 GiB; result = totalMem / 2^30 |
| The device's system locale (`Locale.getDefault()`) | the separator symbol: `,` (ru, de, fr, etc.) or `.` (en_US and others) |

Notation by magnitude:

| totalMem (example) | GiB | On the wire (ru locale) | On the wire (en locale) |
|---|---|---|---|
| 4 294 967 296 (4 GiB) | 4.0 | `4` | `4` |
| 3 758 096 384 (3.5 GiB) | 3.5 | `3,5` | `3.5` |
| 3 823 104 000 (SM-A325F) | 3.5625 | `3,56` | `3.56` |
| 1 610 612 736 (1.5 GiB) | 1.5 | `1,5` | `1.5` |
| 8 589 934 592 (8 GiB) | 8.0 | `8` | `8` |

Important for reproduction: the `#.##` format **never produces** two digits when the second one is zero, and produces no fractional part for whole GiB. Strings like "4,00" or "3,50" are impossible from the real application.

## Examples

- Live capture (SM-A325F, ru, Wi-Fi without SIM): `device_ram=3,56` — identical in /v2/code and /v2/register.
- A 2 GiB device with an en locale: `device_ram=2`.
- A 6 GiB device (5.78 available in totalMem) with a de locale: `device_ram=5,78`.
