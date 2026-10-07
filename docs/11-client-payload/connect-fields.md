# ClientPayload connection fields: 10, 12, 13, 15, 16, 43

Build 2.26.35.75 (263507522). The connection fields are set by two fillers: `X.1J8`
(DI case 6559: shortConnect, connectReason) and `X.1J1` (DI case 6558: connectType,
dnsSource, connectAttemptCount, connectionSequenceInfo). Code: `X/C1US.java:20027-20382`.

## shortConnect (field 10, bool)

Constant `SHORT_CONNECT_FIELD_NUMBER = 10`. Set always (`X/C1US.java:20346-20351`):

```java
boolean zA03 = ((C0SN) interfaceC001600s2.get()).A03();
c1hs6.bitField0_ |= 64;
c1hs6.shortConnect_ = zA03;
```

The logic of `X/C0SN.java:35-38`:

```java
return (prefs.A0R().A02().contains("c2dm_reg_id")
        || !TextUtils.isEmpty(prefs.A0R().A02().getString("fbns_token", null)))
    && prefs.A0T().A02().getInt("logins_with_messages", 0) < 3;
```

So `shortConnect=true` only when **a push token already exists** (Google
c2dm/Firebase `c2dm_reg_id` or Facebook `fbns_token`) **and** the app has still
logged in "with messages" fewer than 3 times. The point: after the phone powers on,
WhatsApp makes a short connect for pushes without holding the socket. On first login
there is no token → `shortConnect=false`; the same for a clean installation without
Google services. The value is visible in the `handshake_payload short_connect=` log
(`X/C0SW.java:1650`).

## connectType (field 12, enum C1J2)

Constant `CONNECT_TYPE_FIELD_NUMBER = 12`. The `X.1J1` filler
(`X/C1US.java:20069-20129`) takes the network snapshot `C06890Tx` (original `X.0Tx`)
from `AnonymousClass078.A0N()`:

```java
C06890Tx tx = ((AnonymousClass078) ...).A0N();
if (tx == null)                     c1j2 = C1J2.CELLULAR_UNKNOWN;
else if (tx.A07)                    c1j2 = C1J2.WIFI_UNKNOWN;        // A07 = wifi
else if (tx.A05) {                  // A05 = mobile
    switch (tx.A00) { ... }         // A00 = NetworkInfo.getSubtype()
} else                              c1j2 = C1J2.CELLULAR_UNKNOWN;
c1hs.connectType_ = c1j2.getNumber();
```

`C06890Tx` (`X/C06890Tx.java:81-90`) is a wrapper over `android.net.NetworkInfo`:
`A07` = isWifi, `A05` = isMobile, `A00` = getSubtype (`TelephonyManager.NETWORK_TYPE_*`
codes), `A04` = connected, `A06` = roaming.

Complete mapping table (phone subtype → `X/C1J2.java` enum value on the wire):

|_subtype (Tx.A00)| Radio | connectType | Number in protobuf |
|---|---|---|---|
| — | no NetworkInfo / neither wifi nor mobile | `CELLULAR_UNKNOWN` | 0 |
| — | Wi-Fi (A07) | `WIFI_UNKNOWN` | 1 |
| 1 | GPRS | `CELLULAR_GPRS` | 104 |
| 2 | EDGE | `CELLULAR_EDGE` | 100 |
| 3 | UMTS | `CELLULAR_UMTS` | 102 |
| 4 | CDMA | `CELLULAR_CDMA` | 108 |
| 5, 6, 12 | EVDO (0/A/B) | `CELLULAR_EVDO` | 103 |
| 7 | 1XRTT | `CELLULAR_1XRTT` | 109 |
| 8 | HSDPA | `CELLULAR_HSDPA` | 105 |
| 9 | HSUPA | `CELLULAR_HSUPA` | 106 |
| 10 | HSPA | `CELLULAR_HSPA` | 107 |
| 11 | IDEN | `CELLULAR_IDEN` | 101 |
| 13 | LTE | `CELLULAR_LTE` | 111 |
| 14 | EHRPD | `CELLULAR_EHRPD` | 110 |
| 15 | HSPAP | `CELLULAR_HSPAP` | 112 |
| other | unknown subtype | `CELLULAR_UNKNOWN` | 0 |

Important consequences: Wi-Fi never yields 0 (it is always 1); "no data" and "could
not determine" are not distinguished — both `CELLULAR_UNKNOWN`(0); zero means
precisely "unknown cellular", not "no network". Live example: the first-login capture
went over Wi-Fi (phone without SIM) → `connectType = WIFI_UNKNOWN` (1); in the HTTP
capture of the same installation the parameter `network_radio_type=1` — the same one.

## connectReason (field 13, enum C1JA)

Constant `CONNECT_REASON_FIELD_NUMBER = 13`. The `X.1J8` filler
(`X/C1US.java:20357-20380`):

```java
C0SO c0soA00 = ((C0SN) ...).A00();            // снимок прошлого обрыва
c1hs8.connectReason_ = C1JA.UNKNOWN.getNumber();   // дефолт 6, пишется всегда
if (c0soA00.A00 != 0) {                       // код прошлого обрыва
    long j = c1hm.A03;                        // loginStartTime, мс
    long j2 = c0soA00.A02;                    // время прошлого обрыва, мс
    if (j2 <= 0 || j - j2 >= 10_000L) return; // обрыв старше 10 секунд — не считается
    int i2 = c0soA00.A00;
    if (i2 == 1)      c1ja = C1JA.USER_ACTIVATED;   // 1
    else if (i2 == 2) c1ja = C1JA.PUSH;             // 0
    else return;                              // любой другой код — остаётся UNKNOWN
    c1hs9.connectReason_ = c1ja.getNumber();
}
```

The full enum `X/C1JA.java`: PUSH 0, USER_ACTIVATED 1, SCHEDULED 2, ERROR_RECONNECT 3,
NETWORK_SWITCH 4, PING_RECONNECT 5, UNKNOWN 6 — but the filler can physically write
only three: UNKNOWN (default), USER_ACTIVATED, and PUSH. The default is overwritten
only when the previous disconnect was **less than 10 seconds ago** and its code in
`C0SO.A00` is 1 (the user opened the app themselves) or 2 (the client was woken by
a push).

`C0SO`/`C0SN` (`X/C0SN.java:12-33`): `A02()` zeroes the code and time, `A01()`
increments the `A01` counter (not used in the payload), `A00()` returns the snapshot.
On **first login** there is no previous disconnect (`A00==0`) → the payload always
carries `UNKNOWN` (6). Visible in the `handshake_payload connect_reason=` log.

## dnsSource (field 15, message C1J4)

Constant `DNS_SOURCE_FIELD_NUMBER = 15`. The nested message `X/C1J4.java`:

- field 15 `dnsMethod` (enum `C1J3`): SYSTEM 0, GOOGLE 1, HARDCODED 2, OVERRIDE 3,
  FALLBACK 4, MNS 5, MNS_SECONDARY 6, SOCKS_PROXY 7;
- field 16 `appCached` (bool).

The filler (`X/C1US.java:20130-20233`) takes `C1HM.A05` — `C26581Fm`
(dnsResolverInfo: int `A00` + bool `A01`) and maps the internal resolver code:

| C26581Fm.A00 | dnsMethod |
|---|---|
| 0 | SYSTEM |
| 1 | GOOGLE |
| 2 | HARDCODED |
| 3 | OVERRIDE |
| 4 | FALLBACK |
| 5, 6 | MNS |
| 7 | MNS_SECONDARY |
| 8 | SOCKS_PROXY |

`appCached` = `C26581Fm.A01` — whether the last address was obtained from the
application cache ("dns from cache after a network change"). The source of
dnsResolverInfo is the DNS mechanism chosen in `C0SW` while preparing the address
(MNS — Meta Name Service, the second code — its backup instance; SOCKS_PROXY appears
with a proxy transport).

## connectAttemptCount (field 16, int32)

Constant `CONNECT_ATTEMPT_COUNT_FIELD_NUMBER = 16`. Written always
(`X/C1US.java:20234-20238`):

```java
int i13 = c1hm.A00;               // attemptedSuccessfulConnections
c1hs5.bitField0_ |= 1024;
c1hs5.connectAttemptCount_ = i13;
```

`C1HM.A00` is the attempt counter of the current connection loop in `C0SW` (the
`this.A00` field of ConnectionThread; in the log `handshake_payload
connect_attempt_count=`). The first attempt after the decision to connect is 0/1
depending on the loop implementation; each failed iteration (DNS, TCP, Noise)
increments it, and the next iteration reassembles ClientPayload with the already
larger value. The field and dnsSource are written by the same `X.1J1` filler, so on
the wire they are always consistent.

## connectionSequenceInfo (field 43, uint32, bit packing)

Constant `CONNECTION_SEQUENCE_INFO_FIELD_NUMBER = 43`. The `X.1J1` filler
(`X/C1US.java:20163-20216` and a repeat at 20239-20292 — jadx duplicates the block,
the value is one). Inputs: `C1FP` connectionMetadata (`c1hm.A08`), `C1HM.A01`
sequenceStep, `C0AT.A01`, `AnonymousClass078.A0P()`.

If hostType `C1FP.A05 == 12` (address not from the host list) — the field is written
as zero. Otherwise a mask is assembled:

```java
i10 = ((i9 & 3) << 0)      // порт
    | ((i6 & 7) << 2)      // тип хоста (network match)
    | (i2 << 5)            // флаг C0AT.A01
    | (i8 << 7)            // sequenceStep 0..31
    | ((vpn ? 1 : 0) << 12);
```

| Bits | Mask | What is encoded | Source and values |
|---|---|---|---|
| 0-1 | 3 | port | `C1FP.A06` (port): 80→0, **443→1**, **5222→2**, other→3 |
| 2-4 | 7 | host category | `AbstractC26591Fn.A00(C1FP.A05)` (see below), then compression to 0..4 |
| 5 | 1 | internal flag | `!C0AT.A01 ? 1 : 0` (volatile flag `X/C0AT.java`) |
| 7-11 | 31 | sequenceStep | `C1HM.A01`, must be 0..31, otherwise `IllegalArgumentException "Counter must be in range 0-31"` |
| 12 | 1 | VPN | `AnonymousClass078.A0P()` — `NetworkCapabilities.hasTransport(4)` (TRANSPORT_VPN), API>=29, see `X/AnonymousClass078.java:130` |

Host category compression: `AbstractC26591Fn.A00` (`X/AbstractC26591Fn.java`) maps
the internal hostType to 1..6 (`{2,3,4}→1, {1,5,9}→2, 8→3, {13,14}→4, {6,10}→5,
{7,11}→6`, everything else → null), then the filler compresses: 1→0, 2→1, 4→2, 5→3,
6→4, null→1, others→1. Human-readable category names are in `C1FP.A03()`:
`push_overrides`, `primary`, `push_fallback`, `fallback`, `hardcoded`, `ex`,
everything else `other`.

Example: host "primary" (category 2→bit value 1), port 443 (1), sequenceStep 2,
no VPN: `(1&3) | (1&7)<<2 | 2<<7 = 1 | 4 | 256 = 261`.

## Live values of the first login (capture)

| Field | Value | Why |
|---|---|---|
| shortConnect | false | no push token yet (the `<iq push>` right after success got 404 item-not-found) |
| connectType | WIFI_UNKNOWN (1) | Wi-Fi without SIM |
| connectReason | UNKNOWN (6) | no previous disconnect |
| dnsSource | depends on the loop's resolver | determined by `C0SW` when choosing the address |
| connectAttemptCount | 0 (first iteration) | the `connect_attempt_count=` log |
| connectionSequenceInfo | port 443/5222 + first attempt's sequenceStep | unpack per the table above |

## Where to look

- `X/C1US.java:20027-20294` (X.1J1), `20295-20382` (X.1J8).
- Enums: `X/C1J2.java`, `X/C1JA.java`, `X/C1J3.java`; message `X/C1J4.java`.
- Network: `X/C06890Tx.java`, `X/AnonymousClass078.java` (`A0N`, `A0P`).
- Disconnects: `X/C0SN.java`, `X/C0SO.java`.
- Connection metadata: `X/C1FP.java`, `X/AbstractC26591Fn.java`.
- Live-value log: `X/C0SW.java:1643-1655` (`handshake_payload ...`).
