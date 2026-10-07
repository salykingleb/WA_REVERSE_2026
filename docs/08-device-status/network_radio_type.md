# network_radio_type — active network connection type (radio interface)

## Purpose

Tells the server which radio interface the device is connected to the network through: Wi-Fi, a specific generation type of mobile network (EDGE/UMTS/LTE/5G NR etc.), or no connection. The numeric codes 100..116 are internal WhatsApp constants mapping TelephonyManager mobile network types; 1 is the Wi-Fi literal.

## Place in the flow

| Request | Presence |
|---|---|
| /v2/code | yes |
| /v2/register | yes |
| /v2/exist | yes |

Placed by the shared collector `IF6.A0O()` (`X/IF6.java:1144–1146`) — the value is identical in all requests collected in the same network state (recomputed anew for each request; it rarely changes between the code and register of a session).

## Format

A number-string from the range **{-1, 0, 1, 100..116}**:

```
network_radio_type=1        ← живой пример (Samsung SM-A325F, Wi-Fi без SIM)
network_radio_type=111      ← мобильное соединение LTE (самый частый мобильный случай)
```

## How it is formed in the application

`X/IF6.java:1144–1146` (inside A0O):

```java
String strValueOf = String.valueOf(
    AbstractC87073t6.A0J(AnonymousClass193.A00(anonymousClass078.A0N())));
map.put("network_radio_type", strValueOf);
```

The chain:

1. `AnonymousClass078.A0N()` — a wrapper of the system network provider; returns `NetworkInfo` (android.net) for the active connection. `null` if the network provider is not initialized.
2. `AnonymousClass193.A00(NetworkInfo)` (`X/AnonymousClass193.java`) — mapping NetworkInfo to an internal radio technology enum type: the `C06890Tx` wrapper, where `A05` = mobile (TYPE_MOBILE), `A07` = wifi (TYPE_WIFI), `A00` = the telephony subtype (`NetworkInfo.getSubtype()`).
3. `AbstractC87073t6.A0J(...)` — the enum into a numeric code. The enum constant names are obfuscated by renaming into constants of the `C26106Bda` class (the digits 100..116 are reassigned redex identifiers).

## Value selection conditions (full table)

| Value | Network state | TelephonyManager type / condition |
|---|---|---|
| **-1** | NetworkInfo == null | network provider not initialized |
| **0** | no connection / unknown | the mobile type GSM (16), LTE_CA (19) and other unlisted ones |
| **1** | **Wi-Fi** | TYPE_WIFI (literal) |
| 100 | EDGE | subtype 2 |
| 101 | IDEN | subtype 11 |
| 102 | UMTS (3G) | subtype 3 |
| 103 | EVDO_0 / EVDO_A / EVDO_B | subtype 5, 6, 12 |
| 104 | GPRS (2G) | subtype 1 |
| 105 | HSDPA | subtype 8 |
| 106 | HSUPA | subtype 9 |
| 107 | HSPA | subtype 10 |
| 108 | CDMA | subtype 4 |
| 109 | 1xRTT | subtype 7 |
| **110** | EHRPD | subtype 14 |
| **111** | **LTE (4G)** | subtype 13 — the dominant mobile case |
| **112** | HSPAP (HSPA+) | subtype 15 |
| **113** | **NR (5G)** | subtype 20 |
| 115 | IWLAN | subtype 18 |
| 116 | TD-SCDMA | subtype 17 |

Practical frequencies: 1 (Wi-Fi) and 111 (LTE) cover the majority of real requests; 113 (5G) — a growing share of modern devices; 100–110/112/115/116 — rare legacy types.

## Combinations with other fields (device physics)

The value is not independent — it is a derivative of the same radio module state as mcc/cellular_strength/sim_type:

- `network_radio_type=111` (LTE) with `mcc=000` — a contradiction: an LTE connection always yields a registered network operator.
- `network_radio_type=1` (Wi-Fi) + `sim_type=1` — normal (SIM present, data over Wi-Fi), cellular_strength is then usually 0..4 (SIM on the network) or 5.
- A Wi-Fi-only device without a SIM: radio=1 + sim_type=0 + mcc/mnc=000 + cellular_strength=5 — the live dump of the SM-A325F is exactly this profile.
- `network_radio_type=0` for an actually sent request is practically impossible: no connection — no HTTP request either.

## Examples

- Live dump (SM-A325F, Wi-Fi, without SIM): `network_radio_type=1` in both requests.
- A smartphone with a SIM on an LTE network: `network_radio_type=111`, the mcc/mnc of the real operator.
- A modern flagship in a 5G zone: `network_radio_type=113`.
