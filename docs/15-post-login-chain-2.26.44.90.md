# Post-login chain of 2.26.44.90 — nodes, parameters and their origin

The chain below is the outgoing traffic of a fresh registration login (SM-A325F, Android 13,
2.26.44.90 / 264409011) from `<success>` to the steady state. Every node family that was
observed in the session is listed; repeated requests are counted rather than enumerated.

---

## 0. `<success>` (IN)

The server confirms the Noise handshake.

```java
// X.1t8 — login session state
String t = successNode.A0K("t");          // server time, seconds
session.A02 = Long.parseLong(t);          // server timestamp
session.A01 = clock.A03() / 1000;         // local time at success
store.A0u(session.A02);                   // server-time correction for the clock
```

`X.08q` (clock) keeps `currentTimeMillis() - drift`, so every later `t` is server-corrected.

---

## 1. Push config — `iq` / `config` (OUT)

```xml
<iq to="s.whatsapp.net" xmlns="urn:xmpp:whatsapp:push" type="get" id="1">
  <config version="1"/>
</iq>
```

Builder `X.CA0.A00(...)` (`X/CA0:20`). `version` is the literal `"1"`; `id` comes from the
global IQ counter `X.0Rx.A0G()` = `"0" + Integer.toHexString(counter++)`.

```java
new C0T8("config", new C0T5("version", "1"));
```

No gates. The stanza is skipped only when the push token is unavailable.

---

## 2. `ib` / `unified_session` (OUT)

```xml
<ib><unified_session id="127334115"/></ib>
```

Builder `X.0Rt` (`X/C06270Rt:24-42`). The id is derived from the client clock:

```
id = (System.currentTimeMillis() + 259_200_000) % 604_800_000
```

```java
c06270Rt.A06 = String.valueOf((C018008q.A00(c06270Rt.A04) + 259200000) % 604800000);
c06270Rt.A01++;   // count of recomputations
```

The value is cached and recomputed when the MeManager callback or the analytics id-cache
advances. The same formula reproduces the captured value exactly:
`(1794915225206 + 259200000) % 604800000 = 128025206`.

No gates. The `id` attribute is omitted when the profile is not logged in; the `<ib>` child
is then dropped by the caller.

---

## 3. `iq` `w:m` / `media_conn` (OUT)

```xml
<iq xmlns="w:m" type="set" to="s.whatsapp.net" id="01">
  <media_conn last_id="49829133"/>
</iq>
```

Builder `X.0Ur` (`X/C07020Ur:142-160`). `last_id` is the `id` of the previous server
`<media_conn>` reply, persisted in SharedPreferences `route_selector_prefs` under the key
`media_conn` as JSON (`X.1u9`, `X/C42351u9:204,215`).

```java
String lastId = routing == null ? null : routing.A0B;          // C07020Ur:142
C0T8 mediaConn = new C0T8("media_conn",
        lastId != null ? new C0T5[]{new C0T5("last_id", lastId)} : null);
```

AB 149 (`A14(149)`) gates both the read and the prefs write. `last_id` is omitted when the
routing info is unknown.

---

## 4. ACS v1 — `iq` `privatestats` / `sign_credential version=1` (OUT)

```xml
<iq xmlns="privatestats" type="get" to="s.whatsapp.net" id="05">
  <sign_credential version="1">
    <blinded_credential>32 bytes</blinded_credential>
  </sign_credential>
</iq>
```

The v1 node is built inside `com.whatsapp.wamsys.JniBridge.jnidispatchIO` case 7
(`JniBridge:826-845`) and has no Java caller: the 32-byte blinded credential is produced in
the native layer before the JNI upcall, and Java only enforces the length.

```java
C0T3 iq = new C0T3("iq");
iq.A02(new C0T5("xmlns", "privatestats"));
iq.A02(new C0T5("type", "get"));
iq.A02(new C0T5("to", "s.whatsapp.net"));
C0T3 sc = new C0T3("sign_credential");
sc.A02(new C0T5("version", "1"));
C0T3 bc = new C0T3("blinded_credential");
C0T4.A04(bArr2, 32L, 32L);
bc.A01 = bArr2;
```

The response is parsed by `X.KOQ.CYK` (`X/KOQ:59-70`): `t`, `signed_credential`,
`acs_public_key`, optional `dleq_proof{c,s}`; the result is passed back with
`jvidispatchIOOOO(4, wajCtx, signed, acsPub, configId)`.

---

## 5. ToS — `iq` `tos` / `request` / `notice` (OUT)

```xml
<iq xmlns="tos" type="get" to="s.whatsapp.net" id="06">
  <request><notice id="20210210"/></request>
</iq>
```

Builder `X.2Er.run()` (`X/RunnableC49192Er:27-32`); the notice list lives in `X.1ti` as
`singletonList("20210210")`. Each entry becomes a `<notice id>`, and `iq@id` comes from
`X.0Rx.A0G()`.

```java
List A0A = C01i.A0A("20210210");           // X.1ti.A0A
new C0T8("notice", new C0T5("id", entry));
new C0T8("request", notice);
```

Gates: AB-int 908 (`A0Z(908)`) against the pref `tos_fetch_iteration` for the first fetch,
then the refresh interval `request_refresh_rate_seconds` (default 86 400 000 ms); plus the
registration check `X.09F.BhC()` and the acceptance-state filter `X.1uY.A00(id)`.
The stanza is absent for an empty list, when offline, or when every id was refreshed recently.

---

## 6. Prekey upload — `iq` `encrypt` (OUT)

```xml
<iq id="2" xmlns="encrypt" type="set" to="s.whatsapp.net">
  <identity>32 bytes</identity>
  <registration>4 bytes</registration>
  <type>05</type>
  <list>
    <key id="0bb469"><value>32 bytes</value></key>
    …
  </list>
  <skey id="59d197"><value>32 bytes</value><signature>64 bytes</signature></skey>
</iq>
```

Builder: the writer-thread handler in `X.0zX` (message 85, `X/HandlerThreadC22800zX:40`); the
payload object is `X.EbK`, filled by `X.17m` MyPreKeysManager (`X/C249517m:348-413`).

| Element | Origin |
|---|---|
| `<identity>` | Signal store, table `identities`, self row `(recipient_id=-1, recipient_type=0, device_id=0)`, column `public_key`; created once at first login by `F9V.A01()` |
| `<registration>` | `identities.registration_id`, random `1..2147483646` (`SecureRandom("SHA1PRNG")`), serialised 4-byte big-endian |
| `<type>` | literal `0x05` — Curve25519 key type (`e_keytype`) |
| `<list>/<key>` | all unsent one-time prekeys, hard cap 812, id = 3-byte BE counter `((n+id) % 16777214)+1`, value = 32-byte Curve25519 public key; the stored `0x05` prefix is stripped before sending |
| `<skey>/@id` | newest `signed_prekeys.prekey_id` (random 24-bit) |
| `<skey>/<value>` | signed-prekey Curve25519 public key |
| `<skey>/<signature>` | XEdDSA, 64 bytes, over `0x05 ‖ signedPreKeyPublic`, signed with the identity private key |

```java
// identity
byte[] identity = this.A01.A04().A01.A00.A01;            // raw 32 B
// registration id
return new byte[]{(byte)(i>>24),(byte)(i>>16),(byte)(i>>8),(byte)i};
// one-time prekey value
byte[] pub = prekey.A00().A01.A00();                     // 33 B with 0x05
System.arraycopy(pub, 1, value, 0, pub.length - 1);
// skey signature
C40551qn.A00("best").A04(identityPriv, signedPreKeyPublic.A00());
```

`<op mode="set">` appears only under AB 23538; PQ prekeys are gated by AB 21201 (count),
21198 and 24160. The 812 cap is hard-coded.

---

## 7. `ib` / `keystore_attestation` (OUT)

```xml
<ib><keystore_attestation>3420 bytes DER</keystore_attestation></ib>
```

Builder `X.17x.A04` (`X/17x:2327-2400`), challenge in `X.17x.A00` (`X/17x:1315-1421`),
sender `X.2Eu` (`X/2Eu:895-1080`).

The challenge is `uint64BE(seconds) ‖ 0x1F ‖ p2`, where `p2` is 32 bytes supplied by the
caller; the `SecureRandom(AB 2078)` branch is used only when `p2 == null`.

```java
ByteBuffer bb = ByteBuffer.allocate(p2.length + 9).order(BIG_ENDIAN);
bb.putLong(clock.A00() / 1000);
bb.put((byte) 0x1F);
bb.put(p2);
specBuilder.setAttestationChallenge(bb.array());
for (int i = chain.length - 1; i >= 0; i--) out.write(chain[i].getEncoded());
```

Gates: AB 1934 (`A07()`), AB 1935 (sender), throttle AB 2067 with the pref
`ka_stanza_sent_ts`, alias AB 2074, random length AB 2078, key validity window AB 2079.

---

## 8. `ib` / `gpia` / `jws` (IN request, OUT response)

```xml
<!-- server -->
<ib><gpia><request nonce="…"/></gpia></ib>
<!-- client -->
<ib><gpia><jws>…</jws></gpia></ib>
```

Builder `X.16Z` (`X/C16Z:16-51`): `AsA()` returns message 254, calls
`jvidispatchIOOOO(3, nonce, …)` and sends the `ib/gpia/jws` node with id 371. The value is a
Play Integrity JWS bound to the server nonce.

---

## 9. `ib` / `integrity_payload` (OUT)

```xml
<ib><integrity_payload>1024 base64 characters</integrity_payload></ib>
```

The node is assembled natively and only wrapped by Java (`JniBridge:887-903`). The payload is
a 768-byte blob `base64(IV[16] ‖ AES-256-CBC(JSON))` with the key
`SHA256(std_base64(authkey))`; the JSON keys are
`aid, _dh, _lh, _iln, _ln, _ge, _gp, _ga, _ip, _isb, _n, _icr, _is`.

The stanza is sent in response to `<safetynet><integrity nonce>`, with no local gate.

---

## 10. Push registration — `config` with `num_acc`, `platform` (OUT)

```xml
<iq xmlns="urn:xmpp:whatsapp:push" type="set">
  <config num_acc="2" platform="gcm"/>
</iq>
```

Builder `X.CA0.A00(...)` (`X/CA0:20`).

| Attribute | Origin |
|---|---|
| `num_acc` | client-side account count: pref `number_of_inactive_accounts` + 1 (`X.0Qd.A0B()`), or `accounts.size() + 2` on the add-account path; validated `0..5` |
| `platform` | push transport literal: `"gcm"` (FCM), `"fbns"`, `"adm"`; validated against `{gcm, adm, fbns}` |

```java
if (numAcc != null && C0T4.A05(numAcc, 0L, 5L, false)) {
    iq.A07("num_acc", ..., numAcc);
}
iq.A07("platform", "gcm", Arrays.asList("gcm", "adm", "fbns"));
```

The whole config is skipped when the push token is unavailable; `num_acc` is omitted when
the value is null or outside `0..5`.

---

## 11. Account bootstrap queries (OUT)

### 11.1 `get_user_disclosures`

```xml
<get_user_disclosures t="1794915236"/>
```

Builder `X.FC3.AFF()` case 1 (`X/FC3:80-97`). `t = clock.A00() / 1000`, clamped by
`(v > 2^31 || v <= 0) ? 100 : (int) v`. No gate inside; the work is scheduled only when the
local disclosure record is missing.

### 11.2 `w:b` / `lists`

```xml
<iq xmlns="w:b" type="get" to="s.whatsapp.net" id="…"><lists/></iq>
```

Builder `X.8KK.run()` case 27 (`X/C8KK:354-363`). `<lists/>` carries no parameters; the
trigger is the pref `refresh_broadcast_lists == true`, reset to `false` after a successful
reply, so at most one fetch per pending refresh.

### 11.3 `newsletter` / `my_addons`

```xml
<newsletter><my_addons limit="1000"/></newsletter>
<newsletter><my_addons type="status" limit="100"/></newsletter>
```

Builder `X.E3V.<init>(jid, id, type, limit)` (`X/E3V:287-311`). The `jid` attribute is added
only for a non-null newsletter jid; `limit` is 1000 (jid null) or 300 (jid set), and 100 for
the status variant; `type="status"` is used only by the status job. `limit` is validated
`0..5000`.

---

## 12. `presence type="available"` (OUT)

Builder `X.1t8.A01()` (`X/C41731t8:61-66`).

```java
C0T3 p = new C0T3("presence");
p.A06("available", "type", Arrays.asList("available", "unavailable"));
((C41301sR) this.A07).DE6(p.A01(), 1);
```

No gates. On connect it is sent only for a new session when the caller flag is set; the twin
message sends `"unavailable"`.

---

## 13. `sync_results` (OUT)

```xml
<sync_results eval_ts_sec="1794915238"/>
```

Builder `X.J7C` (`X/J7C:149-170`), `smax_id = 144`. `eval_ts_sec` is the server timestamp of
the gap-enforcement evaluation (`EvaluationResults.timestamp` in ms), divided by 1000 and
emitted only when positive. The child list is limited to 30 elements.

---

## 14. `iq` `w:stats` / `add` (OUT, 21 batches in the session)

```xml
<iq xmlns="w:stats" type="set" to="s.whatsapp.net" id="02">
  <add t="1794914537">WAM\x05… binary</add>
</iq>
```

The node is built in `com.whatsapp.wamsys.JniBridge.jnidispatchIO` case 6
(`JniBridge:795-808`). The body is the native WAMSys fieldstats buffer handed over JNI
(magic `WAM\x05`); `t = System.currentTimeMillis() / 1000`, validated to
`1577865600..4102473600`; the body length is validated to `4..126976` bytes.

Gates: AB 9654, plus the call-state checks AB 15507 and 468.

---

## 15. `w:comms` (OUT)

### 15.1 `w:comms:chat` → `query/blocking_status`

Builder `X.1h4.A0M` (`X/1h4:6754-6961`). Gate AB 1844 (`A14(1844)`); the stanza is sent with
transport type 340 and only on the 403-NACK recovery path or after a successful
change-number.

### 15.2 `w:comms` → `event`

```xml
<event promotion_id="1248258327353925" timestamp_sec="1794915239"
       logdata="{&quot;nux_id&quot;:&quot;11114&quot;}" type="impression"/>
```

Builder `X.7B`, constructor with `$t=5` (`X/J7B:200-228`).

- `promotion_id` — promotion id from the QP model, 1..30 characters;
- `timestamp_sec` — `System.currentTimeMillis()/1000`, within 2020..2100;
- `logdata` — JSON of the QP logging map (`nux_id`, `instance_log_data`);
- `type` — one of `impression`, `exposure`, `dismiss`, `primary_click`, `secondary_click`,
  `view_entrypoint`, `notification_received`;
- AB 19330 selects the GraphQL/Falco route; the XML event is the fallback.

### 15.3 `optoutlist`

`$t=1` (get, optional `item dhash`) is gated by AB 12758; the set variant carries
`item{@jid, @dhash, @signup_id, @duration, @action, @entry_point}`, with `dhash` up to 100
characters and `signup_id` up to 64.

---

## 16. `ent:silent_nonce`

There is no client-side builder for this node in 2.26.44.90. The literal exists only as
`X.ED1.ACCOUNT_RECOVERY_SILENT_NONCE` (type string, ordinal 42) and in the class-id map
`X.EPQ` (class 276). It is an inbound notification type; the client only sends the automatic
`<ack class="notification" type="ent:silent_nonce" …>` echo.

---

## 17. ACS v2 — `sign_credential version=2` (OUT)

```xml
<iq xmlns="privatestats" type="get" to="s.whatsapp.net" id="035">
  <sign_credential version="2">
    <blinded_credential>32 bytes</blinded_credential>
    <project_name>WA_StatusMusic</project_name>
  </sign_credential>
</iq>
```

Builder `X.1XB.A00` (`X/C1XB:35-94`); blinding in `X.1XD.A03` (`X/1XD:1281-1975`).

- token: `token_length` random bytes from `SecureRandom("SHA1PRNG")`, with `token_length`
  from AB 966 (default 32);
- blinding factor: 32 random bytes with `r[31] &= 0x1F`, rejection-sampled against the
  Ed25519 group order and non-zero;
- `blinded_credential`: VOPRF blind (curve25519, or Ed25519 VOPRF for the project
  allow-list `X.1Ws.A0A`);
- `project_name`: one of the 28 configured names (`WA_BizDirectorySearch`, `WA_PrivateStats`,
  `WA_StatusMusic`, `WA_Theia`, …);
- `config_id`: optional, from the server key-config JSON, omitted when null.

ACS parameters are gated by AB 966, 965, 967–972; AB 17191 selects the send API, AB 17275
enables prefetch. The request is sent only while XMPP is connected.

---

## 18. `w:mex` — `trace/flow_id` + `query` (OUT, 36 queries in the session)

```xml
<iq xmlns="w:mex" type="get" to="s.whatsapp.net" id="0a">
  <trace><flow_id>28529461280028767</flow_id></trace>
  <query query_id="28529461280028767">{"queryId":"28529461280028767","variables":{…}}</query>
</iq>
```

Builders: `X.0tz` for `trace` (`X/C19480tz:8-16`) and `X.0u1` for `query`
(`X/C19500u1:22-29`).

`flow_id` and `query_id` are the same value: the persisted per-query MEX id resolved by
`X.0nP.A00(callName, configName)` from the server MEX config. The body is
`{"queryId": <id>, "variables": <params>}` (`X.0t8`).

Limits: `flow_id` 1..36 characters, `query_id` 1..20, body 1..8 448 000; transport gates are
AB 21936, the keys `whatsapp-android-mex` / `-www` / `-facebook-schema`, and AB 20160 / 20162.
Queries routed to Pando GraphQL bypass this XML path.

---

## 19. Other stanza families of the same session

The session contains 22 distinct `xmlns` families. The ones listed above are the login and
account-bootstrap core; the remaining families are:

| xmlns | Count | Content |
|---|---|---|
| `status` | 3 | `iq id=3 xmlns=status type=set` with a `<status>` child; `privacy` get |
| `blocklist` | 3 | `<iq … xmlns=blocklist type=get><item dhash=…/></iq>` |
| `w:p` | 3 | `iq type=get xmlns=w:p to=s.whatsapp.net` |
| `usync` | 2 | contact sync (see below) |
| `urn:xmpp:whatsapp:account` | 2 | `<crypto action="create">` with a 32-byte `google` key and `t` |
| `w:auth:backup:token` | 2 | `<token>20 bytes</token><type>blockstore` / `foa</type>` |
| `w:profile:picture` | 1 | `<iq … target="222583156932850" type="get"><picture type="preview"/></iq>` |
| `passive` | 1 | `<iq id=4 xmlns=passive type=set><active type="available"/></iq>` |
| `disappearing_mode` | 1 | `iq type=get xmlns=disappearing_mode` |
| `w:biz` | 1 | `<mobile_config name="biz_report_reasons" v="1">` |

### 19.1 Contact sync (`usync`)

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

`sid` is one sync session id per pass (`ContactSyncHelperKt/sync_sid_multi_iq_<uuid>`);
`mode=full`, `index=0`, `last=true` describe a single-chunk full sync. The first pass uses
`context=registration`, a later one uses `context=background` and adds
`allow_mutation="true"`. Contacts are addressed by LID, not by phone number.

### 19.2 Account crypto and backup tokens

```xml
<iq xmlns="urn:xmpp:whatsapp:account" type="get" id="01d">
  <crypto action="create"><google>32 bytes</google><t>1794915236</t></crypto>
</iq>
<iq xmlns="w:auth:backup:token" type="set" id="02a">
  <token>20 bytes</token><type>blockstore</type>
</iq>
```

`action=create` initialises the account-level crypto state; the backup-token stanzas register
the 20-byte `blockstore` and `foa` tokens.
