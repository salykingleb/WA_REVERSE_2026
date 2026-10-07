# simnum, sim_type, sim_state, read_phone_permission_granted — SIM card and phone permissions state

## Purpose

Four related SIM module status parameters:

- **simnum** — "whether the device has a readable phone number on the SIM" (not "the number of SIMs"!). Sent in /v2/code and /v2/register.
- **sim_type** — the generalized SIM state: physical SIM present / absent / telephony unavailable. Sent in /v2/code and /v2/register.
- **sim_state** — the "raw" SIM state per TelephonyManager (0..5). Computed by the application, but sent ONLY in /v2/exist — it is NOT present in /v2/code and /v2/register.
- **read_phone_permission_granted** — whether the READ_PHONE_STATE permission has been granted. Also only /v2/exist, not sent in code/register.

The most important property: simnum, sim_type, sim_state, and read_phone_permission_granted are computed from the same radio module state and the same permission policy, so within a session the values in /v2/code and /v2/register (for the fields that are sent) are always consistent — the shared collectors `IF6.A0O`/`A0R` work on top of the same APIs.

## Place in the flow

| Parameter | /v2/code | /v2/register | /v2/exist |
|---|---|---|---|
| simnum | yes | yes | — (not in the exist map) |
| sim_type | yes | yes | — |
| sim_state | **NO** | **NO** | yes |
| read_phone_permission_granted | **NO** | **NO** | yes |

## Format (+live example)

All four are number-strings (UTF-8 bytes):

```
simnum=0
sim_type=0
```

Live capture of 2026-09-22 (Samsung SM-A325F, Wi-Fi WITHOUT SIM): `simnum=0`, `sim_type=0` — in BOTH requests (/v2/code and /v2/register); `sim_state` and `read_phone_permission_granted` are absent on the wire in code/register (in this session /v2/exist was not analyzed, but that is exactly where they belong).

## How it is formed in the application

### simnum — `X/IF6.java:1146–1148` (inside A0O)

```java
String strA00 = I9Q.A00(application, B0Z.A0a(if6.A0U), AbstractC49572Fv.A0g(if6.A0P));
if (strA00 != null) { z = strA00.length() >= 6; }
map.put("simnum", z ? "1" : "0");
```

`I9Q.A00()` (`X/I9Q.java:22–55`) — reading the device's phone number:
1. The phone permission is checked (helper `C06740Ti.A0I()` — READ_PHONE_STATE). No permission → `null` (log "verifynumber/getphonennumber/permission denied").
2. With the permission: `SubscriptionManager.from(context).getActiveSubscriptionInfoList()` — iterating over active subscriptions, returning the **first non-empty** `SubscriptionInfo.getNumber()`.
3. Fallback: `TelephonyManager.getLine1Number()` (on error — logged, null).
4. Returning the string or null.

Then in `A0O`: string != null AND length >= 6 characters → `"1"`, otherwise `"0"`. The threshold of 6 was chosen as the minimum length of a valid number (the same constant is used in `IEz.A00` when comparing numbers).

### sim_type — `X/IF6.java:530–543` (inside A0R, gated by AB-prop 4435)

```java
TelephonyManager tm = B0W.A0a(...).A0K();   // обёртка getSystemService(TELEPHONY_SERVICE)
if (tm == null)                     i = 2;
else { i = 1; if (tm.getSimState() == 1) i = 0; }   // 1 = SIM_STATE_ABSENT
map.put("sim_type", String.valueOf(i));
```

Android API: `TelephonyManager.getSimState()`. The entire `A0R` group (sim_type, airplane_mode_type, cellular_strength, roaming_type) is added only if AB flag 4435 is enabled.

### sim_state — only /v2/exist, `X/IF6.java:1330–1337` (inside A0j)

```java
TelephonyManager tm = ...A0K();
int simState = (tm != null) ? tm.getSimState() : -1;
map.put("sim_state", String.valueOf(simState));   // put в карту exist: IF6.java:1394/1542
```

### read_phone_permission_granted — only /v2/exist, `X/IF6.java:1329` (inside A0j)

```java
boolean zA0I = B0Z.A0a(this.A0U).A0I();   // phone-permission helper: выдано ли READ_PHONE_STATE
map.put("read_phone_permission_granted", zA0I ? "1" : "0");   // put: IF6.java:1393/1541
```

## Value selection conditions

### simnum

| Value | Condition |
|---|---|
| `"1"` | READ_PHONE_STATE permission granted AND the SIM number was read (via SubscriptionManager or getLine1Number) AND its length >= 6 characters |
| `"0"` | No permission; no SIM; the operator does not return the MSISDN (getNumber() empty — a frequent case even with a SIM!); the number is shorter than 6 characters |

### sim_type

| Value | Condition |
|---|---|
| `"0"` | `TelephonyManager.getSimState() == 1` — SIM_STATE_ABSENT: SIM missing/not visible (Wi-Fi-only device, empty slot) |
| `"1"` | SIM in any state EXCEPT ABSENT (READY, PIN_REQUIRED, PUK_REQUIRED, NETWORK_LOCKED, UNKNOWN-with-SIM) — "physical SIM present" |
| `"2"` | `TelephonyManager == null` — telephony unavailable at all (exotic: devices without a telephony stack) |

### sim_state (exist only)

| Value | TelephonyManager constant | Meaning |
|---|---|---|
| `-1` | — | TelephonyManager == null |
| `0` | SIM_STATE_UNKNOWN | state not determined |
| `1` | SIM_STATE_ABSENT | no SIM |
| `2` | SIM_STATE_PIN_REQUIRED | waiting for PIN |
| `3` | SIM_STATE_PUK_REQUIRED | waiting for PUK |
| `4` | SIM_STATE_NETWORK_LOCKED | locked to an operator |
| `5` | SIM_STATE_READY | ready (the norm) |

### read_phone_permission_granted (exist only)

| Value | Condition |
|---|---|
| `"1"` | READ_PHONE_STATE permission granted to the application |
| `"0"` | Not granted (denied or not requested yet) |

### Combinations (device physics — contradiction markers)

- `sim_type=0` (no SIM) ⇒ `simnum=0` and `sim_mcc/sim_mnc=000`, cellular_strength usually 5.
- `simnum=1` ⇒ SIM present ⇒ `sim_type` must be 1.
- `sim_state` in /v2/code or /v2/register — impossible: the application does not put it there.

## Examples

- **Live capture (SM-A325F, Wi-Fi without SIM, without a granted phone permission)**: `simnum=0`, `sim_type=0` in both requests; sim_state / read_phone_permission_granted absent in code/register.
- A regular phone with a SIM and a granted READ_PHONE_STATE, the operator returns the MSISDN: `simnum=1`, `sim_type=1`.
- A phone with a SIM, but the operator hides the number (typical for a number of MVNOs): `simnum=0`, `sim_type=1`.
