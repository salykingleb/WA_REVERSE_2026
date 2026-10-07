# network_operator_name / sim_operator_name — network and SIM operator names

## Purpose

Human-readable operator names, as returned by the Android radio stack:

- `network_operator_name` — the name of the currently registered network
  (`TelephonyManager.getNetworkOperatorName()`) — the brand whose signal the phone picks up
  right now;
- `sim_operator_name` — the SIM's home operator name (SPN,
  `TelephonyManager.getSimOperatorName()`).

A logical pair to mcc/mnc/sim_mcc/sim_mnc: the codes are "who it is in digits", the names
are "what it is called". Both fields are sent ONLY in `/v2/register` (and `/exist`); in
`/v2/code` they are ABSENT. When there is no data (no SIM/no network) the app puts an EMPTY
STRING rather than omitting the fields or synthesizing a substitute.

## Place in the flow

Collected in `IF6.A0I` (the /v2/register map, `X/IF6.java:340-385`): lines 344-361 —
reading the TelephonyManager (the same block as for mcc/mnc/sim_*), lines 379-380 — puts
both fields:
```java
map.put("network_operator_name", AbstractC87053t4.A1b(networkOperatorName, charset));
map.put("sim_operator_name",     AbstractC87053t4.A1b(str3, charset));
```
The duplicate block for `/exist` — `IF6.A0j` (`X/IF6.java:1339-1353`, put on 1395-1396).
In `A0H` (the /v2/code map, IF6.java:1107-1132) operator names are neither read nor put —
there are no such fields in /v2/code at all.

Wire order in /v2/register (live dump): `sim_operator_name` — 9th position (right after
`_gg`), `network_operator_name` — 21st (after `aid`, before `mcc`).

## Wire format (+live example)

UTF-8 string → bytes (`AbstractC87053t4.A1b`) → percent-encoded query (`HSJ.A00`, %XX in
UPPERCASE): non-Latin names produce sequences like `%D0%...`; Latin brands pass through as
is. "No data" — an empty string (`network_operator_name=`), the field IS PRESENT on the
wire.

Live capture 2026-09-22 (Samsung SM-A325F, Wi-Fi WITHOUT SIM, /v2/register):

```
sim_operator_name=&network_operator_name=
```

(both strings are present with an empty value; in the same session
mcc=000/mnc=000/sim_mcc=000/sim_mnc=000, sim_type=0, simnum=0). Example of a device with a
SIM in the home network: `network_operator_name=MegaFon&sim_operator_name=MegaFon` (the
SIM's SPN matches its own operator's network name).

## How it is formed in the app

`X/IF6.java:341-361` (method `A0I`, /v2/register):

```java
TelephonyManager tm = ...A0K();
String networkOperatorName;
String str3 = Voip.REJECT_REASON_DECLINED;          // "" — Voip.java:54
if (tm == null || (networkOperatorName = tm.getNetworkOperatorName()) == null) {
    networkOperatorName = Voip.REJECT_REASON_DECLINED;   // null → ""
    if (tm != null) {
        String simOperatorName = tm.getSimOperatorName();
        if (simOperatorName != null) str3 = simOperatorName;
    }
} else {
    String simOperatorName = tm.getSimOperatorName();
    if (simOperatorName != null) str3 = simOperatorName;
}
...
map.put("network_operator_name", ...);   // IF6.java:379
map.put("sim_operator_name", ...);       // IF6.java:380
```

That is:
- `network_operator_name` = `tm.getNetworkOperatorName()`, with `tm == null` or `null` from
  the API → `""` (the constant `Voip.REJECT_REASON_DECLINED`,
  `com/whatsapp/calling/voipcalling/Voip.java:54`);
- `sim_operator_name` = `tm.getSimOperatorName()`, with null → `""`.

There is NO fallback concatenation ("MCC 7 MNC 002"), dictionaries, or placeholders
("unknown") in the app — the value is either a real name from the radio stack or an empty
string. The presence of a synthetic string on the wire is a marker that a live client
cannot reproduce.

## Value selection conditions (all variants)

| Device state | network_operator_name | sim_operator_name | Where on the wire |
|---|---|---|---|
| SIM present, home network | the operator's network name ("MegaFon", "MTS", "Vodafone", "AT&T"…) | the same name (the SIM's SPN = its own operator's network name) | /v2/register only (+ /exist) |
| SIM present, roaming | the GUEST network's name (may differ from the SIM) | the home operator's name | /v2/register only |
| SIM present, no network registration (Wi-Fi) | `""` (getNetworkOperatorName null/empty with live sim_mcc/sim_mnc) | the home operator's name | /v2/register only |
| No SIM (Wi-Fi without SIM) | `""` | `""` | /v2/register only — live dump |
| TelephonyManager == null | `""` | `""` | /v2/register only |
| MVNO / virtual operator | the MVNO brand name, may not match the host network | the SIM's SPN | /v2/register only |
| /v2/code (any state) | field ABSENT | field ABSENT | — |

Consistency with the operator codes: the name and the codes are read in ONE block from one
TelephonyManager, so for a live client `network_operator_name=""` is combined only with
mcc/mnc=`000` (no network), and `sim_operator_name=""` — only with sim_mcc/sim_mnc=`000`
(no SIM). The combination `sim_mnc=002` + `sim_operator_name=""` practically never occurs
on a real device with a SIM (anomalous).

## Examples

- Live dump /v2/register (Wi-Fi without SIM): `sim_operator_name=`,
  `network_operator_name=` (empty, positions 9 and 21 in the wire order; in /v2/code these
  fields do not exist).
- RU, MegaFon, home network: `network_operator_name=MegaFon`,
  `sim_operator_name=MegaFon` + `mcc=250&mnc=002&sim_mcc=250&sim_mnc=002`.
- Roaming of a RU-SIM in a Vodafone UK network: `network_operator_name=Vodafone`,
  `sim_operator_name=MegaFon` + `mcc=234&mnc=15&sim_mcc=250&sim_mnc=002` (+ roaming_type=1).
- A Wi-Fi device with a SIM, no network: `network_operator_name=` (empty) with a non-empty
  `sim_operator_name=MegaFon` and `mcc=000&mnc=000&sim_mcc=250&sim_mnc=002`.
- A name with spaces/non-Latin characters is percent-encoded: `Beeline%20RU`,
  `МТС` → `%D0%9C%D0%A2%D0%A1` (the real wire form — whatever the radio stack returns).
