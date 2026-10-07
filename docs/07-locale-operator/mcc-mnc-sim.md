# mcc / mnc / sim_mcc / sim_mnc — network and SIM operator codes

## Purpose

The four fields tell the server the device's cellular identity:

- `mcc` / `mnc` — the Mobile Country Code / Mobile Network Code of the **registered
  (current) network** operator — where the phone is connected right now
  (`TelephonyManager.getNetworkOperator()`);
- `sim_mcc` / `sim_mnc` — the same for the **SIM card's home operator**
  (`TelephonyManager.getSimOperator()`).

The difference between the pairs is the main signal: in the home network they match; a
mismatch = roaming (mcc/mnc — the guest country's network, sim_mcc/sim_mnc — the home
operator). `000` in all four fields — a device without SIM/not registered in a network
(typically: a tablet/Wi-Fi).

Sent in `/v2/code` and `/v2/register` (in `/v2/exist` — the same values via the common
collector). Between requests of the same session the values match byte for byte (live
dump).

## Place in the flow

- `/v2/code`: `RequestCodeRepository$requestCode$2.java:197-198` reads the TelephonyManager
  and parses both strings BEFORE assembling the map; then `IF6.A0H`
  (`X/IF6.java:1107-1132`, call to `A0K(c1Id, c1Id2, ...)` on line 1118) puts the 4 fields
  into the request map.
- `/v2/register`: `IF6.A0I` (`X/IF6.java:340-385`): lines 344-346 — the same two calls,
  line 378 — the same `A0K`.

Wire order (native, the fields are spread apart):
- `/v2/code`: `sim_mnc` (5th), `mnc` (7th), `mcc` (18th), `sim_mcc` (26th);
- `/v2/register`: `sim_mnc` (3rd), `mcc` (21st), `mnc` (23rd), `sim_mcc` (27th).

## Wire format (+live example)

All four are exactly 3 ASCII digits, each field individually; leading zeros are preserved.

Live capture 2026-09-22 (Samsung SM-A325F, Wi-Fi WITHOUT SIM):

```
mcc=000&mnc=000&sim_mcc=000&sim_mnc=000
```

("no data" is encoded as `"000"`, not as an empty string and not as a missing parameter).
Example of a device with a MegaFon SIM in the home network:
`mcc=250&mnc=002&sim_mcc=250&sim_mnc=002` (exactly `002`, not `02` — see the formatting
below). Roaming: `mcc=234&mnc=15&sim_mcc=250&sim_mnc=002`.

## How it is formed in the app

The source is two TelephonyManager calls, each returning ONE combined string
`mcc+mnc` (5–6 digits), which the app parses with the `C1Id` parser:

- `RequestCodeRepository$requestCode$2.java:197-198` (for /v2/code) and `X/IF6.java:344-346`
  (for /v2/register, method `A0I`):
  ```java
  TelephonyManager tm = ...A0K();
  C1Id net = C1Id.A00(tm != null ? tm.getNetworkOperator() : null);  // → mcc/mnc
  C1Id sim = C1Id.A00(tm != null ? tm.getSimOperator() : null);      // → sim_mcc/sim_mnc
  ```
- Parser `X/C1Id.java:11-53` (`C1Id.A00`):
  ```java
  Pattern A02 = Pattern.compile("(\\d{3})(\\d{2,3})");   // полное совпадение 5-6 цифр
  // mcc  = группа 1 КАК ЕСТЬ → всегда 3 цифры;
  // mnc  = String.format(Locale.US, "%03d", Integer.valueOf(group2));
  //        «02»→«002», «67»→«067», «260»→«260» — ВСЕГДА ровно 3 цифры с ведущими нулями;
  // null / пусто / не матчит / NumberFormatException → new C1Id("000","000")
  ```
- Put into the map by `IF6.A0K` (`X/IF6.java:411-428`) in the order
  `mcc, mnc, sim_mcc, sim_mnc` (UTF-8 bytes → percent-encoded; for digits the encoding is
  an identity).

The wire format is a consequence of the parser:
- Android `getNetworkOperator()` returns the raw concatenation of mcc+mnc, where mnc is
  3 digits in North America and often 2 digits in the rest of the world. The app NEVER
  sends a 2-digit mnc: the `%03d` reformatting guarantees 3 digits (idempotent:
  "002"→"002").
- "No operator" — getNetworkOperator() returns an empty string (no network registration) or
  getSimOperator() is empty (no SIM) → the regex does not match → `"000"/"000"`.

## Value selection conditions (all variants)

| Device state | mcc / mnc (network) | sim_mcc / sim_mnc (SIM) | Comment |
|---|---|---|---|
| SIM present, registered in the home network | real values (e.g. 250/002) | the same values | the pairs match — the norm |
| SIM present, roaming | mcc/mnc = the GUEST country's network | sim_* = the home operator | mcc≠sim_mcc — the roaming marker; consistent with roaming_type=1 |
| SIM present, no cellular network registration (Wi-Fi only) | `000`/`000` (getNetworkOperator empty) | real sim_* | the split "network unknown, SIM known" |
| No SIM (tablet/Wi-Fi phone) | `000`/`000` | `000`/`000` | live dump; consistent with sim_type=0, simnum=0, network_radio_type=1 (Wi-Fi), cellular_strength=5 |
| TelephonyManager == null (edge case) | `000`/`000` | `000`/`000` | `tm != null ? ... : null` in both calls |
| The device itself has no effect | — | — | mcc/mnc do NOT depend on cc/in (the number's country code) or on lg/lc (locale) |

Missing parameters on the wire are not provided for: even with a complete absence of data
the fields are present with the value `000` (put unconditionally in `A0K`).

The difference between mcc and sim_mcc (summary):
1. different API sources: `getNetworkOperator()` vs `getSimOperator()`;
2. home network: equal; roaming: different (mcc — where the device is, sim_mcc — whose SIM
   it is);
3. Wi-Fi without registration: mcc=000 with live sim_* (the empty string parser → 000, the
   SIM is read);
4. server semantics: the consistency of mcc↔the number's cc and mcc↔sim_mcc (0/equality) —
   verifiable invariants; the pair "mcc of country A + cc of country B" is possible for a
   live client only in roaming (and then mcc≠sim_mcc), otherwise it is a nonexistent
   combination.

## Examples

- Live dump (Wi-Fi without SIM): `mcc=000`, `mnc=000`, `sim_mcc=000`, `sim_mnc=000` — match
  byte for byte in code and register.
- Russia, MegaFon, home network: `mcc=250`, `mnc=002`, `sim_mcc=250`, `sim_mnc=002`
  (the operator's internal key "02" → "002" on the wire).
- Russia, Beeline: `mnc=099` (raw "99" → `%03d` → "099").
- USA, AT&T: `mcc=310`, `mnc=070` (American mnc are 3-digit — "070" stays "070").
- Roaming: RU-SIM in a UK network: `mcc=234`, `mnc=15`, `sim_mcc=250`,
  `sim_mnc=02`→`sim_mnc=002`.
- Wi-Fi tablet with a SIM, not registered: `mcc=000`, `mnc=000`, `sim_mcc=250`,
  `sim_mnc=002`.
