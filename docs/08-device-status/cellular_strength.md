# cellular_strength — cellular network signal level

## Purpose

Tells the server the cellular network signal level (0–4 on the standard Android SignalStrength scale) or "no data" (5). Important: **5 is not "excellent signal" but the absence of information** about the signal — typical for devices without a SIM / without service, connected over Wi-Fi.

## Place in the flow

| Request | Presence |
|---|---|
| /v2/code | yes |
| /v2/register | yes |
| /v2/exist | yes |

Placed by the collector `IF6.A0R()` (`X/IF6.java:545`) — the entire A0R group (sim_type, airplane_mode_type, cellular_strength, roaming_type) is added only with AB flag 4435 enabled (enabled in current builds; the live dump contains the field).

## Format

A number-string from the range **0..5**:

```
cellular_strength=5        ← живой пример (Samsung SM-A325F, Wi-Fi без SIM)
```

## How it is formed in the application

`X/IF6.java:545` (inside A0R):

```java
map.put("cellular_strength", String.valueOf(
    (Build.VERSION.SDK_INT < 28
        || (telephonyManager = B0W.A0a(...).A0K()) == null
        || telephonyManager.getSignalStrength() == null)
        ? 5
        : telephonyManager.getSignalStrength().getLevel()));
```

Three sources of "no data" (→ 5):

1. `Build.VERSION.SDK_INT < 28` — on Android below 9.0 `SignalStrength.getLevel()` does not exist at all (added in API 28), the application does not risk calling it;
2. `TelephonyManager == null` — telephony unavailable;
3. `TelephonyManager.getSignalStrength() == null` — the signal object has not been obtained yet: **no SIM / no network registration / radio off** (exactly this case for the Wi-Fi-only profile: SignalStrength null with working Wi-Fi).

Otherwise — `SignalStrength.getLevel()` (Android API 28+): the aggregated level 0..4, the same one the system signal icon shows.

## Value selection conditions

| Value | Meaning | Condition |
|---|---|---|
| **5** | NO DATA (unknown) | SDK < 28, OR TelephonyManager null, OR SignalStrength null (no SIM/network — a frequent case on Wi-Fi devices) |
| 0 | no signal | SIM on the network, but the level is minimal/absent (underground, far cell) |
| 1 | very weak | |
| 2 | weak | |
| 3 | good | |
| 4 | excellent | |

The key nuance: the default "unknown" state is encoded exactly as 5, not -1 and not an empty string. On a device without a SIM (even with working Wi-Fi) the value is always 5.

Combinations: with `network_radio_type=111` (LTE connection active) the value 5 is impossible (SignalStrength is not null); with `sim_type=0` (no SIM) the value 5 is practically guaranteed.

## Examples

- Live dump (SM-A325F, Wi-Fi without SIM): `cellular_strength=5` in both requests — SignalStrength null, no mobile service.
- A smartphone in a confident LTE coverage zone: `cellular_strength=4` or `3`.
- A smartphone at the edge of a coverage zone: `cellular_strength=1`.
