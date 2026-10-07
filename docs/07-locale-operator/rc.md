# rc — release channel

## Purpose

`rc` — the build distribution channel number (release channel).
The value is constant for the entire app lifecycle and equals `0` for any store
(Play Store) build of WhatsApp.

Present in ALL registration requests: `/v2/code`, `/v2/register`, `/v2/exist`, `/v2/consent`,
and it is also added again by the native bridge (see below). In the code and register of one
session the value is the same.

## Place in the flow

Added when assembling each parameter map via the common method `IF6.A0O`
(`X/IF6.java:1139-1158`, call to `A0Z(map)` on line 1156) — that is, in one block with
`network_radio_type`, `simnum`, `hasinrc`, `pid`. It is also called separately from other
maps (the `A0H`/`A0I`/`A0j` paths, code/register/exist). Wire order:
- `/v2/code`: `..., network_radio_type, lg, rc, pid, ...`
- `/v2/register`: `..., lg, rc, network_radio_type, pid, ...`

## Wire format (+live example)

A string — the decimal representation of a number: `"0"`, `"1"`, `"2"` or `"3"`. UTF-8 bytes
(`B0W.A1Z`) → percent-encoded query (for a digit the encoding is an identity).

Live capture 2026-09-22 (store build 2.26.35.75):

```
rc=0
```

in /v2/code and /v2/register — the same.

## How it is formed in the app

`X/IF6.java:649-657` (`IF6.A0Z`):

```java
public static final void A0Z(java.util.Map map) {
    C1Io c1Io = C1Io.RELEASE;                    // захардкожено RELEASE
    String string = Integer.valueOf(c1Io.getNumber()).toString();  // "0"
    if (string == null) {
        string = Voip.REJECT_REASON_DECLINED;     // "" — недостижимо
    }
    map.put("rc", B0W.A1Z(string));
}
```

Channel enum `X/C1Io.java:7-11` (protobuf `Internal.EnumLite`):

```java
public enum C1Io {
    RELEASE(0), BETA(1), ALPHA(2), DEBUG(3);
    ...
}
```

There is no channel selection logic in the Java layer — the `RELEASE` value is written
literally. The channel is determined by where the build was installed from (store = RELEASE,
beta program = BETA, etc.), but for client requests it is always a constant of the specific
APK.

Additionally: the native bridge `IF6.A0S` (`X/IF6.java:553-567`,
`JniBridge.jvidispatchOOO(16, app, wajContext)`) merges its own set of parameters from
`libs.so` into the map — the composition is obfuscated, but rc-like common fields are
duplicated there as well. On the wire this is indistinguishable: a single `rc` key with the
value `"0"`.

## Value selection conditions (all variants)

| Variant | rc value | When it occurs |
|---|---|---|
| Store build (Play Store) | `"0"` (RELEASE) | the vast majority of devices; hardcoded |
| Google Play beta program | `"1"` (BETA) | beta channel |
| Alpha | `"2"` (ALPHA) | internal/closed testing |
| Debug build | `"3"` (DEBUG) | development |
| Retries / repeated code request / repeated register | does NOT change — rc is not a counter | always |
| Wi-Fi without SIM / roaming / any network | `"0"` — does not depend on the network | always |

### Why "0" and what actually counts attempts

- `rc="0"` because the code literally contains `C1Io.RELEASE` with `getNumber() == 0`.
- The real counter of code request attempts lives in ANOTHER parameter — the JSON
  `client_metrics` → field `attempts` (`X/IF6.java:1117`, assembled in `X/H8Z.A01`;
  live dump: code — `"attempts":7`, register — `"attempts":0`). It is this one that grows on
  repeated code requests, not rc.
- There is no storage of rc at all: the value is written neither to SharedPreferences nor to
  files — it is a compile-time enum constant in every `A0Z` call.

## Examples

- Live dump /v2/code: `rc=0`; /v2/register: `rc=0` (match byte for byte).
- Any repeated send of /v2/code after a failure: `rc=0`,
  `client_metrics={"attempts":N,...}` with a grown N — rc stays `0`.
- A hypothetical beta build of the same APK: `rc=1` — to imitate store clients the value
  must always be `0`.
