# ClientPayload: how the encrypted login payload is assembled

WhatsApp build 2.26.35.75, versionCode 263507522. ClientPayload is the protobuf message
`X/C1HS.java` (jadx name `C1HS`, original `X.1HS`) that is sent **encrypted** inside
the Noise handshake: on a full handshake (first login) — in the `payload` field of the
ClientFinish frame (`X/C1LY`), on resume — in the `payload` field of the ClientHello
frame (`X/C1LR`). Encryption is performed by `X/C1KE` (span `encrypt_login_payload`),
so the payload is never seen in plaintext on the wire: the FunXmpp hook in the
first-login capture prints only what goes after `<success>`.

## Who assembles it: C1HN from DI set 6929

The assembly point is `X/C1HN.java` (original name `X.1HN`, "LoginPayloadProvider"):

```java
public final class C1HN {
    public final C05D A00;   // ленивый DI-провайдер
    public final Set A01;    // набор филлеров

    public final C1HS A00(C1HM c1hm) {
        C1Ho c1Ho = (C1Ho) C1HS.DEFAULT_INSTANCE.createBuilder();
        for (C1Hr c1Hr : C09Y.A00(this.A01, AbstractC017208b.A04(((C00W) this.A00.A00.get()).A02(), 6930))) {
            C000800h.A09(c1Ho);
            c1Hr.AAR(c1hm, c1Ho);
        }
        return (C1HS) c1Ho.build();
    }

    public C1HN() {
        Set setA05 = C00C.A05(6929);      // DI-набор 6929
        C000800h.A06(setA05);
        this.A01 = setA05;
        this.A00 = AnonymousClass057.A00(5);
    }
}
```

Mechanics:

1. The constructor takes the filler set from DI set **6929** (`C00C.A05(6929)`).
2. `A00(C1HM)` creates a `C1Ho` builder from `C1HS.DEFAULT_INSTANCE`.
3. The set is sorted via `C09Y.A00(set, AbstractC017208b.A04(..., 6930))` — the sort
   key is taken from DI set 6930, i.e. the field population order is fixed and does
   not depend on the set iteration order.
4. Each filler is an implementation of the interface `X/C1Hr.java` (original `X.1Hr`)
   with the single method `void AAR(C1HM c1hm, C1Ho c1Ho)`: it appends its fields to
   the shared builder.
5. `C1Ho` is an anonymous subclass of `GeneratedMessageLite.Builder` generated inside
   `C1HS.dynamicMethod` (case `NEW_BUILDER`, jadx comment `// from class: X.1Ho`).
   There is no separate `X/C1Ho.java` file in the jadx output — the builder lives
   inside `C1HS.java`.

The input of the assembly is the context `X/C1HM.java` (`XmppLoginContext`); its
fields (see `toString`, `C1HM.java:43-80`):

| C1HM field | Name in toString | What it is |
|---|---|---|
| `A04` | jid | account UserJid (null during companion registration) |
| `A09` | passive | computed connection passive flag |
| `A02` | sessionId | int, login attempt identifier |
| `A03` | loginStartTime | long, login start in millis |
| `A05` | dnsResolverInfo | `C26581Fm` {A00 = int DNS method, A01 = bool appCached} |
| `A00` | attemptedSuccessfulConnections | attempt number within the connection loop |
| `A06` | companionModeRegParams | `C26191Dt`, companion e-fields (only when jid==null) |
| `A0A` | signalProtocolStoreIsNew | pref `signal_protocol_store_is_new` |
| `A07` | clientQueueState | `C71163Gf` {A00 preacksCount, A01 processingQueueSize} or null |
| `A08` | connectionMetadata | `C1FP` (socket sessionId, hostType A05, port A06) |
| `A01` | sequenceStep | int 0..31 for connectionSequenceInfo |

The context is created by `C0SW` right before the handshake (`C0SW.java:1631-1635`):

```java
boolean z5 = (zA02 || ((C04930Mc) this.A0T.get()).A02()
              || (!z2 && !zA0P && !zA1J && !zA09)) ? false : true;
C71163Gf c71163GfA1S = A1S(z5);
UserJid userJidA06 = c1fb.A06(userJidAom);
C1HS c1hsA00 = ((C1HN) c05dA00.get()).A00(
        c1fb.A07(userJidA06, c26191Dt, c71163GfA1S, this.A00, z5, zA1J));
Log.i("ConnectionThread/connect: SEND <handshake_payload connect_attempt_count=... "
    + "login_count=... passive=... session_id=... short_connect=... connect_type=... "
    + "connect_reason=... /> hasPreacks=... enablePassiveModeBasedOnQueueSize=...");
```

From this log line (`ConnectionThread/connect: SEND <handshake_payload ...>`) you can
see the values of six fields live, before encryption.

## The C1Hr filler set (all are anonymous classes in X/C1US.java)

Each filler is registered in the DI factory `X/C1US.java` and created by case:

| DI case | Original name | What it populates | Lines in C1US.java |
|---|---|---|---|
| 6556 | `X.1Hs` | userAgent (5), oc (23), yearClass (36), memClass (37) | 19805-20011 |
| 6557 | `X.1Iz` | trafficAnonymization (40) | 20012-20026 |
| 6558 | `X.1J1` | connectType (12), dnsSource (15), connectAttemptCount (16), connectionSequenceInfo (43) | 20027-20294 |
| 6559 | `X.1J8` | lidDbMigrated (41), passive (3), processingQueueSize (46), preacksCount (45), sessionId (9), shortConnect (10), lc (24), connectReason (13) | 20295-20382 |
| 6560 | `X.1JU` | device (18), devicePairingData (19) — companion only | 20383-20685 |
| 6561 | `X.1JV` | username (1), product (20), pushName (7) | 20686-20734 |
| 6567 | `X.1JC` | pairedPeripherals (47) | 20745-20789 |

Registration of the factories themselves: case 6562 → `new C1Ie()` (Build fields),
6563 → `new C1HN()`, 6564 → `new C1Ih()` (fdid) — `C1US.java:20735-20740`.

## Complete table of the C1HS protobuf fields

The numbers are `*_FIELD_NUMBER` constants from `X/C1HS.java:19-56`.

| # | Field | Type | Who sets it | When it is absent |
|---|---|---|---|---|
| 1 | `username` | uint64 | `X.1JV`: numeric prefix of the user JID | no JID (companion) — field not set |
| 3 | `passive` | bool | `X.1J8` from C1HM.A09 | always written |
| 5 | `userAgent` | message `C1Ht` | `X.1Hs` | always |
| 6 | `webInfo` | message `C26961Jf` | — | not populated by the Android client (Web branch) |
| 7 | `pushName` | string | `X.1JV` from `C08X.Avj()` | empty profile — field not set |
| 9 | `sessionId` | int32 | `X.1J8` from C1HM.A02 | always |
| 10 | `shortConnect` | bool | `X.1J8` from `C0SN.A03()` | always |
| 12 | `connectType` | enum `C1J2` | `X.1J1` based on `C06890Tx` | always; no NetworkInfo → `CELLULAR_UNKNOWN`(0) |
| 13 | `connectReason` | enum `C1JA` | `X.1J8`; default `UNKNOWN`(6) | always written |
| 14 | `shards` | repeated int32 | — | not populated by this client |
| 15 | `dnsSource` | message `C1J4` {15 dnsMethod, 16 appCached} | `X.1J1` from C1HM.A05 | always |
| 16 | `connectAttemptCount` | int32 | `X.1J1` from C1HM.A00 | always written |
| 18 | `device` | int32 | `X.1JU` from pref `registration_device_id` | phone (BKW=false) — field absent |
| 19 | `devicePairingData` | message `C26971Jg` | `X.1JU` when `c1hm.A04 == null` | regular phone with a JID — the whole block is absent |
| 20 | `product` | enum `C1JY` | `X.1JV` = `WHATSAPP_LID`(4) | set only on LID login |
| 21 | `fbCat` | bytes | — | iOS/FB branch, not set by Android |
| 22 | `fbUserAgent` | bytes | — | same |
| 23 | `oc` | bool | `X.1Hs`, two-stage signature check | always |
| 24 | `lc` | int32 | `X.1J8` from pref `connection_lc` | always; 0 on first login |
| 30 | `iosAppExtension` | enum | — | iOS branch |
| 31 | `fbAppId` | uint64 | — | FB branch |
| 32 | `fbDeviceId` | bytes | — | FB branch |
| 33 | `pull` | bool | — | not populated by this client |
| 34 | `paddingBytes` | bytes | — | not populated |
| 36 | `yearClass` | int32 | `X.1Hs` from `C1Ix.A01` | always |
| 37 | `memClass` | int32 | `X.1Hs` from `C1Iy.A01` | always |
| 38 | `interopData` | message `C26981Jh` | — | interop branch (Messenger) |
| 40 | `trafficAnonymization` | enum `C1J0` | `X.1Iz` | always: STANDARD(1) or OFF(0) |
| 41 | `lidDbMigrated` | bool | `X.1J8` when `c1hm.A04 == null` and `C03580Gs` is ready | phone with a JID — field absent |
| 42 | `accountType` | enum | — | not populated by this client |
| 43 | `connectionSequenceInfo` | uint32 | `X.1J1`, bit packing | always written after the filler |
| 44 | `paaLink` | bool | — | not populated |
| 45 | `preacksCount` | int32 | `X.1J8` from C1HM.A07.A00 | no queue snapshot (A1S returned null) — field absent |
| 46 | `processingQueueSize` | int32 | `X.1J8` from C1HM.A07.A01 | same |
| 47 | `pairedPeripherals` | repeated string | `X.1JC`: `smart_glasses`, `garmin` | no AB 27619 / no linked devices |
| 48 | `testIsolationId` | bytes | — | test builds only |

Protobuf field defaults: strings are initialized to `Voip.REJECT_REASON_DECLINED` = `""`
(`C1HS.java:93`, `com/whatsapp/calling/voipcalling/Voip.java:54`), `shards_` — an empty
`IntArrayList`.

## What is NOT part of the payload on a regular phone

**Device pairing (field 19).** The entire `X.1JU` block (case 6560) with the fields
`e_keytype`/`e_regid`/`e_ident`/`e_skey_id`/`e_skey_val`/`e_skey_sig`/`buildHash`/`deviceProps`
executes only when `c1hm.A04 == null` (`C1US.java:20415`), i.e. when the client has
no user JID — companion (linked device) registration. On a phone with a JID the block
is skipped entirely, including `buildHash` = Base64-decode(`C00L.A04("2.26.35.75")`).

**Prekeys.** The identity and the batch of prekeys are not part of ClientPayload,
neither on first login nor on resume. In the first-login capture they are sent as a
separate IQ after `<success>` (id `2` at 02:48:52.636):

```xml
<iq id='2' xmlns='encrypt' type='set' to='s.whatsapp.net'>
  <identity>eXHfoDtXkeGBZAO0FNjHFTjYo5NXRsulzxr2C2AAaFQ=</identity>
  <registration>J/hw3g==</registration><type></type>
  <list><key><id>uBZd</id><value>...</value></key>...</list>
</iq>
```

**Field 18 (`device`).** Set only when `((C017908i) this.A03.A00.get()).BKW(false)`
— i.e. when the pref `registration_device_id` > 0 (`C017908i.java:814-825`). On a
phone this pref is empty and the field is absent.

## The server side of the payload

The server identifies the account by the pair "`username` from the payload + the
client static key from `ClientFinish.static`" (the same authkey that was registered
at `/v2/register`). All attestations (SafetyNet, Play Integrity, keystore) arrive
after `<success>` and are not part of ClientPayload — see chapter 12.

## Where to look in the code

- Protobuf: `X/C1HS.java` (payload), `X/C1Ht.java` (userAgent), `X/C1I0.java` (appVersion), `X/C1J4.java` (dnsSource).
- Provider: `X/C1HN.java`, interface `X/C1Hr.java`, context `X/C1HM.java`.
- Fillers: `X/C1US.java` cases 6556-6561, 6567.
- Call site and the `<handshake_payload>` log: `X/C0SW.java` around 1631-1660.
- Packing into the Noise frame: `X/C1KE.java` (handshake analysis — chapter 10).
