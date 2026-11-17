# Post-login chain of 2.26.44.90 — every node, its parameters, and how the app builds them

This document is **not** the raw capture. The raw capture is
`hooks/wa_agent_log_registration.txt`:
**6008 lines, 5845 outgoing write events, 110 incoming nodes** (fresh registration,
SM-A325F, 2.26.44.90 / 264409011). It contains 812 prekey elements, 21 `w:stats`
batches and 36 `w:mex` queries — repeated requests are only counted here, while every
**node type** is analysed below.

`W<n>` = anchor into the raw log (`[W<n>] …` lines).

---

## 0. `<success>` (IN)

Server confirms the Noise handshake.

```java
// X.1t8 (jadx C41731t8) — login session state
String t = successNode.A0K("t");          // server time, seconds
session.A02 = Long.parseLong(t);          // server timestamp
session.A01 = clock.A03() / 1000;         // local time at success
store.A0u(session.A02);                   // server-time correction for the clock
```

`X.08q` (clock) keeps `currentTimeMillis() - drift`, so all subsequent `t` values are
server-corrected.

---

## 1. Push config — `iq` / `config` (OUT, W1)

**Wire**

```xml
<iq to="s.whatsapp.net" xmlns="urn:xmpp:whatsapp:push" type="get" id="1">
  <config version="1"/>
</iq>
```

**Builder** — `X.CA0.A00(...)` (jadx `X/CA0.java:20`, body skipped → smali `classes7/X/CA0.smali`).

**Source** — `version` is a hard-coded literal `"1"`; `id` comes from the global IQ counter
`X.0Rx.A0G()` = `"0" + Integer.toHexString(counter++)`.

**Code**

```java
new C0T8("config", new C0T5("version", "1"));
// wrapper: iq{xmlns="urn:xmpp:whatsapp:push", type="get", id=A0G(), to=s.whatsapp.net}
```

**Gates** — none. **Absent** — only if the push token is missing
(`AddAccountNavigator` logs `pushId is null` and skips the send).

---

## 2. `ib` / `unified_session` (OUT, W13)

**Wire**

```xml
<ib><unified_session id="127334115"/></ib>
```

**Builder** — `X.0Rt` (jadx `X/C06270Rt.java:24-42`, send at `:36-42`).

**Source** — the **client clock**, not a random or server id:

```
id = (System.currentTimeMillis() + 259_200_000) % 604_800_000
```

a 7-day modulo shifted by 3 days; cached in a `volatile` field and recomputed when the
MeManager callback or the analytics id-cache advances.

**Code**

```java
c06270Rt.A06 = String.valueOf((C018008q.A00(c06270Rt.A04) + 259200000) % 604800000);
c06270Rt.A01++;   // counter of recomputations
```

Verified numerically: `(1794915225206 + 259200000) % 604800000 = 128025206` — exactly the
value captured in the previous session.

**Gates** — no AB. **Absent** — the `id` attribute is omitted when not logged in
(`X.09F.Bi4() == false`); the whole `<ib>` child is then dropped by the caller.

---

## 3. `iq` `w:m` / `media_conn` (OUT, W88)

**Wire**

```xml
<iq xmlns="w:m" type="set" to="s.whatsapp.net" id="01">
  <media_conn last_id="49829133"/>
</iq>
```

**Builder** — `X.0Ur` (jadx `X/C07020Ur.java:142-160`).

**Source** — `last_id` is the `id` of the previous server `<media_conn id="…">` reply,
persisted as JSON `{"last_id": …}` in SharedPreferences **`route_selector_prefs`** under
key `media_conn` (`X.1u9`, jadx `X/C42351u9.java:204,215`).

**Code**

```java
String lastId = routing == null ? null : routing.A0B;          // C07020Ur.java:142
C0T8 mediaConn = new C0T8("media_conn",
        lastId != null ? new C0T5[]{new C0T5("last_id", lastId)} : null);
```

**Gates** — AB **149** (`A14(149)`) gates both the read and the prefs write.
**Absent** — `last_id` omitted when routing info is unknown (`null` attr array).

---

## 4. ACS v1 — `iq` `privatestats` / `sign_credential version=1` (OUT, W98)

**Wire**

```xml
<iq xmlns="privatestats" type="get" to="s.whatsapp.net" id="05">
  <sign_credential version="1">
    <blinded_credential>32 bytes</blinded_credential>
  </sign_credential>
</iq>
```

**Builder** — `com.whatsapp.wamsys.JniBridge.jnidispatchIO` **case 7**
(jadx `…/JniBridge.java:826-845`). v1 is **native-initiated**: no Java caller exists.

**Source** — `blinded_credential` is produced in `libwhatsapp.so` before the JNI upcall;
Java only enforces the length (`C0T4.A04(bArr, 32, 32)`).

**Code**

```java
C0T3 iq = new C0T3("iq");
iq.A02(new C0T5("xmlns", "privatestats"));
iq.A02(new C0T5("type", "get"));
iq.A02(new C0T5("to", "s.whatsapp.net"));
C0T3 sc = new C0T3("sign_credential");
sc.A02(new C0T5("version", "1"));
C0T3 bc = new C0T3("blinded_credential");
C0T4.A04(bArr2, 32L, 32L);          // exact 32 bytes
bc.A01 = bArr2;
```

**Response** — `X.KOQ.CYK` (jadx `X/KOQ.java:59-70`) parses
`t`, `signed_credential`, `acs_public_key`, optional `dleq_proof{c,s}` and calls
`jvidispatchIOOOO(4, wajCtx, signed, acsPub, configId)`.

**Gates** — none in the builder. **Absent** — never on a healthy session.

---

## 5. ToS — `iq` `tos` / `request` / `notice` (OUT, W103)

**Wire**

```xml
<iq xmlns="tos" type="get" to="s.whatsapp.net" id="06">
  <request><notice id="20210210"/></request>
</iq>
```

**Builder** — `X.2Er.run()` (jadx `X/RunnableC49192Er.java:27-32`, body skipped →
smali `smali2/X/2Er.1.smali:1096-1277`); the notice list lives in `X.1ti`
(`List A0A = singletonList("20210210")`).

**Source** — the list of locally known ToS notice ids; each becomes a `<notice id>`;
`iq@id` from `X.0Rx.A0G()`.

**Code**

```java
List A0A = C01i.A0A("20210210");           // X.1ti.A0A
new C0T8("notice", new C0T5("id", entry));
new C0T8("request", notice);
// iq{to=s.whatsapp.net, id=A0G(), xmlns="tos", type="get"}
```

**Gates** — AB-int **908** (`A0Z(908)`) vs pref `tos_fetch_iteration` for the first fetch;
then a refresh rate `request_refresh_rate_seconds` (default 86 400 000 ms);
`X.09F.BhC()` (registered) and `X.1uY.A00(id)` acceptance-state filter.

**Absent** — empty list, offline (`X.07p.A0V() == false`), or all ids refreshed recently.

---

## 6. Prekey upload — `iq` `encrypt` (OUT, W118)

**Wire**

```xml
<iq id="2" xmlns="encrypt" type="set" to="s.whatsapp.net">
  <identity>32 bytes</identity>
  <registration>4 bytes</registration>
  <type>05</type>
  <list>
    <key id="0bb469"><value>32 bytes</value></key>
    … 812 keys …
  </list>
  <skey id="59d197"><value>32 bytes</value><signature>64 bytes</signature></skey>
</iq>
```

**Builder** — `X.0zX` WriterThread message 85 (jadx `X/HandlerThreadC22800zX.java:40`,
stub → smali `smali1/X/0zX.3.smali:4757-5720`); payload object `X.EbK`
(jadx `X/C32860EbK.java`), filled by `X.17m` MyPreKeysManager
(jadx `X/C249517m.java:348-413`).

**Sources**

| Element | Source |
|---|---|
| `<identity>` | Signal store, table `identities`, self row `(recipient_id=-1, recipient_type=0, device_id=0)`, column `public_key`; generated once at first login by `F9V.A01()` |
| `<registration>` | `identities.registration_id`, random `1..2147483646` (`SecureRandom("SHA1PRNG")`), serialized 4-byte BE |
| `<type>` | literal `0x05` — Curve25519 key type (`e_keytype`) |
| `<list>/<key>` | all unsent one-time prekeys, hard cap **812**, id = 3-byte BE counter `((n+id) % 16777214)+1`, value = 32-byte Curve25519 pubkey (the stored `0x05` prefix is stripped before sending) |
| `<skey>/@id` | newest `signed_prekeys.prekey_id` (random 24-bit) |
| `<skey>/<value>` | signed-prekey Curve25519 pubkey |
| `<skey>/<signature>` | **XEdDSA, 64 bytes, over `0x05 ‖ signedPreKeyPublic`** with the identity private key |

**Code**

```java
// identity: X.0W9.A1E()
byte[] identity = this.A01.A04().A01.A00.A01;     // raw 32B, no 0x05
// registration: X.0W9.A0K.A08() -> AbstractC40401qY.A03(id)
return new byte[]{(byte)(i>>24),(byte)(i>>16),(byte)(i>>8),(byte)i};
// prekey: X.0Xw.A02
byte[] pub = prekey.A00().A01.A00();               // 33B with 0x05
System.arraycopy(pub, 1, value, 0, pub.length - 1); // strip prefix
// skey signature: F9V.A0B
C40551qn.A00("best").A04(identityPriv, signedPreKeyPublic.A00());
```

**Gates** — `<op mode="set">` only when AB **23538** (`0x5bf2`) or optional payload
strings are set; PQ prekeys gated by AB **21201** (count), **21198**/**24160**.
The legacy 812 cap is hard-coded.

**Absent** — never; the store must contain the identity row, otherwise
`Missing entry for self in identities table`.

---

## 7. `ib` / `keystore_attestation` (OUT, W4203)

**Wire** — a single binary child, no attributes:

```xml
<ib><keystore_attestation>3420 bytes DER</keystore_attestation></ib>
```

**Builder** — `X.17x.A04` (chain assembly, `smali1/X/17x.smali:2327-2400`), challenge in
`X.17x.A00` (`:1315-1421`). **Sender** — `X.2Eu` (smali `smali2/X/2Eu.1.smali:895-1080`).

**Source** — Android Keystore certificate chain; challenge is
`uint64BE(seconds) ‖ 0x1F ‖ p2`, where `p2` is **32 bytes supplied by the caller** on this
path (the `SecureRandom(AB 2078)` path is used only when `p2 == null`).

**Code**

```java
ByteBuffer bb = ByteBuffer.allocate(p2.length + 9).order(BIG_ENDIAN);
bb.putLong(clock.A00() / 1000);   // seconds
bb.put((byte) 0x1F);
bb.put(p2);                        // 32 random bytes from the caller
specBuilder.setAttestationChallenge(bb.array());
// …build chain:
for (int i = chain.length - 1; i >= 0; i--) out.write(chain[i].getEncoded());
```

**Gates** — AB **1934** (`A07()`), AB **1935** (sender), throttle AB **2067** with pref
`ka_stanza_sent_ts`, alias AB **2074**, random length AB **2078**, key window AB **2079**.

**Absent** — when the AB gates are off or the chain cannot be obtained
(`blacknoise: got null attestation chain in custom stanza`).

---

## 8. `ib` / `gpia` / `jws` (IN request → OUT response, W4287)

**Wire**

```xml
<!-- server -->
<ib><gpia><request nonce="…"/></gpia></ib>
<!-- client -->
<ib><gpia><jws>…</jws></gpia></ib>
```

**Builder** — `X.16Z` (jadx `X/C16Z.java:16-51`): `AsA()` returns message `254`, calls
`jvidispatchIOOOO(3, nonce, …)` and builds `ib/gpia/jws`, sent with id `371`.

**Source** — Play Integrity JWS bound to the server nonce (native pipeline).

**Gates** — server-initiated only. **Absent** — if the server never requests it.

---

## 9. `ib` / `integrity_payload` (OUT, W4242)

**Wire** — 1024 base64 chars = 768-byte blob:

```xml
<ib><integrity_payload>…</integrity_payload></ib>
```

**Builder** — native; Java only wraps it: `JniBridge` `jvidispatchIOOOO` /
`jvidispatchIO` case, node `ib/integrity_payload` (jadx `JniBridge.java:887-903`).

**Source** — JSON built in `libwhatsapp.so`, keys
`aid, _dh, _lh, _iln, _ln, _ge, _gp, _ga, _ip, _isb, _n, _icr, _is`; encrypted with
`AES-256-CBC`, key `SHA256(std_base64(authkey))`, output `base64(IV[16] ‖ ct)`.

**Gates** — sent in response to `<safetynet><integrity nonce>`; no local AB gate.

**Absent** — if the server does not request integrity.

---

## 10. Push registration — `config` with `num_acc`, `platform` (OUT)

**Wire**

```xml
<iq xmlns="urn:xmpp:whatsapp:push" type="set">
  <config num_acc="2" platform="gcm"/>
</iq>
```

**Builder** — `X.CA0.A00(...)` (jadx `X/CA0.java:20`; smali `classes7/X/CA0.smali:549`).

**Source**

| Attribute | Source |
|---|---|
| `num_acc` | client-side account count: pref `number_of_inactive_accounts` + 1 (`X.0Qd.A0B()`), or `accounts.size() + 2` on the add-account path; validated `0..5` |
| `platform` | push transport literal: `"gcm"` (FCM), `"fbns"`, `"adm"`; validated against `{gcm, adm, fbns}` |

**Code**

```java
if (numAcc != null && C0T4.A05(numAcc, 0L, 5L, false)) {
    iq.A07("num_acc", ..., numAcc);
}
iq.A07("platform", "gcm", Arrays.asList("gcm", "adm", "fbns"));
```

**Gates** — none. **Absent** — when the push token is missing, the whole config is not sent;
`num_acc` omitted if `null` or out of `0..5`.

---

## 11. Account bootstrap queries (OUT)

### 11.1 `get_user_disclosures` (W4571)

```xml
<get_user_disclosures t="1794915236"/>
```

- Builder: `X.FC3.AFF()` case 1 (smali `smali8/X/FC3.smali:944-1147`).
- `t` = `clock.A00() / 1000`, clamped: `(v > 2^31 || v <= 0) ? 100 : (int) v`.
- No AB gate inside; scheduled by WorkManager (`X.143`) only when the local disclosure
  record is missing.

### 11.2 `w:b` / `lists` (W4447)

```xml
<iq xmlns="w:b" type="get" to="s.whatsapp.net" id="…"><lists/></iq>
```

- Builder: `X.8KK.run()` case 27 (jadx `X/C8KK.java:354-363`).
- Source: no parameters — `<lists/>` is empty; the trigger is pref
  `refresh_broadcast_lists == true`, which is reset to `false` after a successful reply
  (so at most one fetch per pending refresh).

### 11.3 `newsletter` / `my_addons` (W4485, W4499)

```xml
<newsletter><my_addons limit="1000"/></newsletter>
<newsletter><my_addons type="status" limit="100"/></newsletter>
```

- Builder: `X.E3V.<init>(jid, id, int type, long limit)` (jadx `X/E3V.java:287-311`).
- `jid` attribute only when the newsletter jid is non-null; `limit` = 1000 (jid null) or
  300 (jid set), 100 for the status variant; `type="status"` only for the status job.
- Bounds: `limit` validated `0..5000`.

---

## 12. `presence type="available"` (OUT, W4764)

**Builder** — `X.1t8.A01()` (jadx `X/C41731t8.java:61-66`).

```java
C0T3 p = new C0T3("presence");
p.A06("available", "type", Arrays.asList("available", "unavailable"));
((C41301sR) this.A07).DE6(p.A01(), 1);
```

**Source** — constant `"available"`. **Gates** — none; sent only for a **new** session when
the caller flag asks for it (the `A0y(..., boolean, boolean)` 4th argument); the twin
message sends `"unavailable"`.

---

## 13. `sync_results` (OUT, W4963)

```xml
<sync_results eval_ts_sec="1794915238"/>
```

- Builder: `X.J7C` (jadx `X/J7C.java:149-170`), `smax_id = 144`.
- Source: **server** timestamp — gap-enforcement `EvaluationResults.timestamp` (ms)
  → `X.2HI.A05(j) = j / 1000` → `eval_ts_sec` (only if `j > 0`).
- No AB gate; child list limited to 30 elements.

---

## 14. `iq` `w:stats` / `add` (OUT, 21 batches)

**Wire**

```xml
<iq xmlns="w:stats" type="set" to="s.whatsapp.net" id="02">
  <add t="1794914537">WAM\x05… binary</add>
</iq>
```

**Builder** — `com.whatsapp.wamsys.JniBridge.jnidispatchIO` **case 6**
(jadx `JniBridge.java:795-808`).

**Source** — the body is the native WAMSys fieldstats buffer handed over JNI
(magic `WAM\x05`); `t = System.currentTimeMillis() / 1000`, validated to
`1577865600..4102473600` (2020-01-01 … 2100-01-01); body length `4..126976` bytes.

**Gates** — AB **9654** (`A14`), plus call-state checks AB **15507** / **468**.

---

## 15. `w:comms` (OUT, W5104)

### 15.1 `w:comms:chat` → `query/blocking_status`

- Builder: `X.1h4.A0M` (smali `smali2/X/1h4.smali:6754-6961`).
- Gate: AB **1844** (`A14(1844)`, const `0x734`); sent as stanza type **340**.
- Fires only on the 403-NACK recovery path and after a successful change-number.

### 15.2 `w:comms` → `event`

```xml
<event promotion_id="1248258327353925" timestamp_sec="1794915239"
       logdata="{&quot;nux_id&quot;:&quot;11114&quot;}" type="impression"/>
```

- Builder: `X.7B` `$t=5` (jadx `J7B.java:200-228`).
- `promotion_id` — QP model field, validated 1..30 chars;
- `timestamp_sec` — `System.currentTimeMillis()/1000`, window 2020..2100;
- `logdata` — JSON of the QP logging map (`nux_id`, `instance_log_data`);
- `type` — one of `impression|exposure|dismiss|primary_click|secondary_click|view_entrypoint|notification_received`;
- Gate: AB **19330** for the GraphQL/Falco route; this XML event is the fallback.

### 15.3 `optoutlist`

- `$t=1` (get, optional `item dhash`) — sender gate AB **12758**;
- `$t` setvariant with `item{@jid, @dhash, @signup_id, @duration, @action, @entry_point}` —
  `dhash` ≤ 100 chars, `signup_id` ≤ 64.

---

## 16. `ent:silent_nonce`

- **No client builder exists** in this build: the literal appears only as
  `X.ED1.ACCOUNT_RECOVERY_SILENT_NONCE` (type string, ordinal 42) and in the class-id map
  `X.EPQ` (class **276**).
- It is an **inbound notification** type; the client only sends the automatic ack
  `<ack class="notification" type="ent:silent_nonce" …>`.

---

## 17. ACS v2 — `sign_credential version=2` (OUT, W5152)

**Wire**

```xml
<iq xmlns="privatestats" type="get" to="s.whatsapp.net" id="035">
  <sign_credential version="2">
    <blinded_credential>32 bytes</blinded_credential>
    <project_name>WA_StatusMusic</project_name>
  </sign_credential>
</iq>
```

**Builder** — `X.1XB.A00` (jadx `X/C1XB.java:35-94`); blinding in `X.1XD.A03`
(smali `smali2/X/1XD.smali:1281-1975`).

**Source**

- token = `token_length` random bytes from `SecureRandom("SHA1PRNG")`
  (`token_length` = AB **966**, default 32);
- blinding factor = random 32 bytes, `r[31] &= 0x1F`, rejection-sampled against the
  Ed25519 group order, non-zero;
- `blinded_credential` = VOPRF blind (curve25519, or Ed25519 VOPRF for the project
  allow-list `X.1Ws.A0A`);
- `project_name` from the hard-coded list (28 names: `WA_BizDirectorySearch`,
  `WA_PrivateStats`, `WA_StatusMusic`, `WA_Theia`, …);
- `config_id` optional, from server key-config JSON (omitted when null).

**Gates** — AB **966/965/967/968/969/970/971/972** (ACS parameters), AB **17191** for the
send API, AB **17275** for prefetch.

**Absent** — only sends while XMPP is connected; `config_id` omitted when unknown.

---

## 18. `w:mex` — `trace/flow_id` + `query` (OUT, 36 queries)

**Wire**

```xml
<iq xmlns="w:mex" type="get" to="s.whatsapp.net" id="0a">
  <trace><flow_id>28529461280028767</flow_id></trace>
  <query query_id="28529461280028767">{"queryId":"28529461280028767","variables":{…}}</query>
</iq>
```

**Builder** — `X.0tz` (trace, jadx `X/C19480tz.java:8-16`) and `X.0u1`
(query, jadx `X/C19500u1.java:22-29`).

**Source** — `flow_id` and `query_id` are the **same** value: the persisted per-query MEX
id resolved by `X.0nP.A00(callName, configName)` from the server MEX config.
The body is `{"queryId": <id>, "variables": <params>}` (`X.0t8`).

**Gates** — `flow_id` 1..36 chars; `query_id` 1..20; body 1..8 448 000;
transport gates AB **21936**, keys `whatsapp-android-mex` / `-www` / `-facebook-schema`,
AB **20160**/**20162**; queries routed to Pando GraphQL bypass this XML path.

---

## 19. Additional stanza families observed in the same burst

The capture contains **22 distinct `xmlns` families**, not only the login-critical ones.
All of them are listed here; the ones already detailed above are referenced, the rest are
described at wire level (their builders were not reverse-engineered to the same depth).

| xmlns | count | What it is | Observed parameters |
|---|---|---|---|
| `w:mex` | 35 | MEX/GraphQL queries | §18 |
| `w:stats` | 21 | WAMSys fieldstats | §14 |
| `tos` | 3 | ToS + disclosures | §5, §11.1 |
| `status` | 3 | status/privacy | `iq id=3 xmlns=status type=set` + `<status>` child; `privacy` get |
| `blocklist` | 3 | blocklist fetch | `<iq … xmlns=blocklist type=get><item dhash=…/></iq>` |
| `w:comms:chat` | 3 | blocking status | §15.1 |
| `w:p` | 3 | privacy/PSA | `iq type=get xmlns=w:p to=s.whatsapp.net` |
| `urn:xmpp:whatsapp:push` | 2 | push config + registration | §1, §10 |
| `privatestats` | 2 | ACS v1 / v2 | §4, §17 |
| `w:b` | 2 | broadcast/business lists | §11.2 |
| `newsletter` | 2 | my_addons | §11.3 |
| `usync` | 2 | **contact sync** | `<usync sid="ContactSyncHelperKt/sync_sid_multi_iq_<uuid>"><index>0</index><last>true</last><mode>full</mode><context>registration\|background</context><query><contact/><addressing_mode>lid</addressing_mode></query></usync>`; the background pass adds `allow_mutation="true"` |
| `urn:xmpp:whatsapp:account` | 2 | account crypto bootstrap | `<crypto action="create">` + 32-byte `google` key |
| `w:auth:backup:token` | 2 | backup token registration | `<token>20 bytes</token><type>blockstore` / `foa</type>` |
| `w:m` | 1 | media connection | §3 |
| `encrypt` | 1 | prekeys | §6 |
| `w:profile:picture` | 1 | profile picture query | `<iq … target="222583156932850" lid=… type=get><picture type="preview"/></iq>` |
| `passive` | 1 | passive-mode activation | `<iq id=4 xmlns=passive type=set to=s.whatsapp.net><active type="available"/></iq>` |
| `disappearing_mode` | 1 | disappearing-mode state | `<iq … xmlns=disappearing_mode type=get>` |
| `w:biz` | 1 | business mobile config | `<mobile_config name="biz_report_reasons" v="1">` |
| `w:comms` | 1 | QP telemetry | §15.2 |
| `optoutlist` | 1 | marketing opt-out | §15.3 |

### 19.1 Contact sync (`usync`)

Two passes are sent right after login:

```xml
<iq to="s.whatsapp.net" xmlns="usync" id="019" type="get">
  <usync sid="ContactSyncHelperKt/sync_sid_multi_iq_d4217284-e406-430f-81c7-f6b314b408e8">
    <index>0</index>
    <last>true</last>
    <mode>full</mode>
    <context>registration</context>
    <query>
      <contact/>
      <addressing_mode>lid</addressing_mode>
    </query>
  </usync>
</iq>
```

- `sid` — `ContactSyncHelperKt/sync_sid_multi_iq_<uuid>` — the sync session id, one per pass.
- `mode=full`, `index=0`, `last=true` — single-chunk full sync.
- `context=registration` for the first pass, `context=background` for the later one; the
  background pass adds `allow_mutation="true"`.
- `addressing_mode=lid` — contacts are addressed by LID, not by phone number.

### 19.2 Account crypto (`urn:xmpp:whatsapp:account`)

```xml
<iq xmlns="urn:xmpp:whatsapp:account" type="get" id="01d">
  <crypto action="create">
    <google>32 bytes</google>
    <t>1794915236</t>
  </crypto>
</iq>
```

- `action=create` — creates the account-level crypto state; `google` is a 32-byte key
  blob; `t` is the server-corrected timestamp.

### 19.3 Backup token (`w:auth:backup:token`)

```xml
<iq xmlns="w:auth:backup:token" type="set" id="02a">
  <token>20 bytes</token>
  <type>blockstore</type>
</iq>
```

- Two tokens are registered: `type=blockstore` and `type=foa` (Facebook account), 20 bytes each.

## Coverage notes

- Raw capture: 6008 lines, 5845 outgoing writes, 110 incoming nodes, 812 prekeys,
  21 `w:stats`, 36 `w:mex`, 2 `presence`.
- Nodes whose builders jadx left as stubs (prekeys `X.0zX`, `X.2Er`, ACS response
  `X.KaO`) were read from baksmali dumps of the corresponding `classes*.dex`.
- Everything above is cross-checked against the capture values where the value is
  deterministic (unified_session formula, prekey count, lengths, attribute sets).
