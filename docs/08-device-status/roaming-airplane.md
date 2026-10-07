# roaming_type, airplane_mode_type — roaming and airplane mode

## Purpose

A pair of network environment status flags:

- **roaming_type** — whether the device is in national/international roaming (`TelephonyManager.isNetworkRoaming()`);
- **airplane_mode_type** — whether airplane mode is enabled on the device (`Settings.Global.AIRPLANE_MODE_ON`).

Both are placed by the single collector `IF6.A0R()` (`X/IF6.java:544–549`) under the gate of AB flag 4435.

## Place in the flow

| Parameter | /v2/code | /v2/register | /v2/exist |
|---|---|---|---|
| roaming_type | yes (conditional — see below) | yes (conditional) | yes (conditional) |
| airplane_mode_type | yes | yes | yes |

A peculiarity of roaming_type: it is placed **only if the network provider is initialized** (`AnonymousClass078.A0N() != null`) — if the application has no NetworkInfo, there is no roaming_type key on the wire at all. airplane_mode_type is placed unconditionally (inside the 4435 gate).

## Format

Number-strings:

```
roaming_type=0
airplane_mode_type=0        ← живой пример: оба "0" (Samsung SM-A325F, Wi-Fi без SIM)
```

## How it is formed in the application

`X/IF6.java:544–549` (inside A0R):

```java
map.put("airplane_mode_type", String.valueOf(
    Settings.Global.getInt(resolver, "airplane_mode_on", 0) != 0 ? 1 : 0));

if (((AnonymousClass078) C05D.A03(if6.A07)).A0N() != null) {     // провайдер сети есть
    TelephonyManager tm3 = B0W.A0a(...).A0K();
    map.put("roaming_type", String.valueOf(
        (int)(tm3 != null ? tm3.isNetworkRoaming() : 2)));
}
```

- **airplane_mode_type**: Android API `Settings.Global.getInt(contentResolver, "airplane_mode_on", 0)` — the system airplane mode setting (1 = enabled). Read directly, default 0.
- **roaming_type**: Android API `TelephonyManager.isNetworkRoaming()` — the roaming flag of network registration. TelephonyManager unavailable → the literal 2.

## Value selection conditions

### roaming_type

| Value | Condition | When it occurs |
|---|---|---|
| **"0"** | not roaming (`isNetworkRoaming() == false`) | the overwhelming majority of registrations (home network) |
| **"1"** | roaming (`isNetworkRoaming() == true`) | the SIM is registered in a foreign network; must be consistent with mcc ≠ sim_mcc |
| **"2"** | TelephonyManager == null | telephony unavailable (exotic) |
| (absent) | network provider not initialized (`A0N() == null`) | rare; in that case the key is not on the wire |

Combination: roaming ⇔ network mcc != sim mcc. With `mcc=000` (no network registration, Wi-Fi profile) roaming physically cannot exist — the live dump gives roaming_type=0 with mcc=000.

### airplane_mode_type

| Value | Condition |
|---|---|
| **"0"** | airplane mode off (`airplane_mode_on` = 0) — the norm; with airplane mode on and no Wi-Fi the request physically could not have been sent |
| **"1"** | airplane mode on (`airplane_mode_on` != 0) — a request with airplane mode + Wi-Fi enabled is theoretically possible |

## Examples

- Live dump (SM-A325F, Wi-Fi without SIM): `roaming_type=0`, `airplane_mode_type=0` — both in /v2/code and /v2/register.
- A phone on a trip (home SIM in mcc=250, network mcc=262): `roaming_type=1`.
- Airplane mode + Wi-Fi: `airplane_mode_type=1`, `roaming_type` — absent or 0, mcc=000.
