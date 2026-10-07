# Login response: `<success>` and `<failure>`, client disconnects before the response

Build 2.26.35.75 (263507522). The finale of the login: after ClientFinish (or the
resume-ClientHello) the socket switches to node framing, and `C0SW.A0y`
reads nodes until it encounters `success` or `failure`. Code: `C0SW.A0y`
(`X/C0SW.java:904-1089`), `C0SW.A14` (`X/C0SW.java:497-517`), `C0SW.A0q`
(`X/C0SW.java:458-465`), `C0SW.A0x` (`X/C0SW.java:885-901`); the login exception
`X/C28861Rq.java`.

## 1. The reading loop

`A0x` (`X/C0SW.java:885-901`) attaches the `C1OV` callback (span `C02S.A0G`) for
reading nodes and leads them through `C1KJ` (span `read_login_response`, code 23).
Parsing — `A0y`: the loop `while (c0p3A07 != null)` over the nodes `C231811k.A07()`:

- allowed tags — `web`, `success`, `failure`;
- `web` — only if the payload contained a web-ref (`c1hs.A04().A00()`), at most
  one; a second one — `C28891Rt("multiple web nodes encountered on login")`;
  with a nested `<error code=...>` the handler calls `C1HI.A0f(code)`
  (`X/C0SW.java:1081-1084`);
- any other tag — `C28891Rt("unexpected node received during login sequence; node=<tag>")`
  (`X/C0SW.java:913-917`);
- end of stream before the response — `C28891Rt("node stream ended unexpectedly")`
  (`X/C0SW.java:1088`).

## 2. `<success>` — attributes

A live example (first login, first-login capture, 02:48:52.424):

```xml
<success t='1788392932' location='vll' pn='6615@s.whatsapp.net'
         lid='4312@lid' abprops='185' creation='1788392926'/>
```

| Attribute | Parsing in the client | Code |
|---------|------------------|-----|
| `t` | Server time, seconds. `c1oo.A02 = Long.parseLong(t)`, `c1oo.A01 = A16.A03()/1000` (local seconds) — the "server/local" pair then measures the clock skew; written to prefs `A0t(...)`. A non-number — `C28891Rt("invalid server time; timeString=...")` | `X/C0SW.java:990-1001` |
| `location` | Datacenter code (not geolocation). Handled by `A0q`: written to prefs only if the string is **shorter than 40 characters** (`TextUtils.isEmpty(strA0K) \|\| strA0K.length() < 40`), then `C28851Rp.A00` and `c018208l.A0I().A07(strA0K)` | `X/C0SW.java:458-465` |
| `pn` | Phone JID; parsed as `PhoneUserJid` and goes into `C42571uZ.A00(...)` | `X/C0SW.java:1070` |
| `lid` | LID. If a different LID existed locally — the telemetry report `lid-chatd-lid-mismatch` (the session is NOT torn down); when there is no local one — `reg-lid-chatd-lid-expected-but-null`; on the first LID the log gets `lid-lifecycle/login-success isFirstLidLogin=... registrationLidMissing=... registrationJidMissing=... passive=... regIdPrefLastWrite/History` and `CQR(jid)` is written | `X/C0SW.java:1011-1069` |
| `props` | The regular props set number → `atomicReference` (a props request later, if the local hash is empty) | `X/C0SW.java:1003-1006` |
| `abprops` | The AB-props set number → `atomicReference2`; with an empty local hash the client then sends `<iq xmlns='abt'><props protocol='1' hash=''/></iq>` — in the capture id 07, hash empty; `abprops=185` is the version number, not the props themselves | `X/C0SW.java:1007-1010` |
| `creation` | The account creation timestamp on the server; the client does **not** read it in `A0y` | — |
| `static_pq_key` | Base64 ML-KEM-512 server key; handled by `A14` (see below) | `X/C0SW.java:497-517` |

`C0SW.A14` (`X/C0SW.java:497-517`): the condition — PQ config at level ENABLE+
(`Ey.A04() == C02S.A00` — disabled, or `Ey.A05() != C02S.A01` — not the
right variant, then exit), the `static_pq_key` attribute present; the body —
`Base64.decode(strA0M, 3)`, log `ConnectionThread/login/success: static_pq_key
received, size=<n>`, saving `AnonymousClass139.A0H(new KEMPublicKey(...))`
(prefs `server_static_pq_public`) — this is the key for the KEM-resume of the next
connection. A base64 error — the telemetry report `noise-pq-static-key-decode-failed`
with `base64_len=<n>`. In the live capture the attribute was absent.

## 3. `<failure reason='...'>` — full table

Reason is a numeric attribute (`c0p3A07.A04("reason")`), log
`ConnectionThread/login/failure/reason=<n>` (`X/C0SW.java:919-923`). With the
companions mode enabled and `C018208l.A1J()` additionally `C28861Ckv.A01(3,
reason)` (`X/C0SW.java:924-926`). Before parsing, `A0q` is also called
on failure (location is saved on failure too, `X/C0SW.java:927`).

| reason | Internal type (`C28861Rq.type`) | Meaning | Attributes read |
|--------|----------------------------------|-------|-------------------|
| 500–599 | 4 | Temporary server failure, the socket can be retried | — (`throw new C28861Rq(4, reason)`, `X/C0SW.java:928-930`) |
| 402 | 13/14/15/2 | Ban/expire. With `appeal_token` and `code`: 109 → 15, otherwise with a token: 106 → 13, 107 → 14; without a token or other codes → 2 | `expire` → `expire_time_out`; `code`; `message` → `banMessage`; `url` → `faqUrl`; `appeal_token` → `banAppealToken`; `age_collection` → `ageCollection` (`X/C0SW.java:931-954`) |
| 403 | 7 | Account blocked by policy | `is_eu` → `isEu`; `vt` → `violationType`; `violation_reason`; `appeal_token`; `reg_info` → `regInfo`; plus logout-button texts from `A13` (`X/C0SW.java:956-962`, `487-495`) |
| 405 | 3 | Client too old | `t` → `expiration_time = A08("t", 0) * 1000` ms (`X/C0SW.java:964-968`) |
| 406 | 5 | A new login / generation change is needed | `code` (`X/C0SW.java:969-973`) |
| 416 | 11 | A violation tied to another account | `vt`; `violation_reason`; `source_acct` → `violationSourceAcct`; `appeal_token` (`X/C0SW.java:974-982`) |
| other | 0 | Unknown failure | logout-button texts via `A13`, if present (`X/C0SW.java:975, 984-985`) |

The logout texts (`A13`, `X/C0SW.java:487-495`) — from the node's attributes:
`logout_message_header`, `logout_message_subtext`, `logout_message_locale`,
`logout_main_button_text/url`, `logout_secondary_button_text/url` (fields of
`C28861Rq`, `X/C28861Rq.java:15-21`). The code inside the type: `serverErrorCode`
= reason, `code` = the `code` attribute.

## 4. Client disconnects before the server response

These errors are created by the client itself, without waiting for the server's
`<failure>`; the catch branches in `C0SW.A0v`:

| Event | Exception | Log in `A0v` | Error type |
|---------|------------|-------------|------------|
| Certificate failed verification (ServerHello chapter) | `C1S0` | `ConnectionThread/connect/socket/invalid-certificate-exception` + `c1fb.A0C()` | 10 (`X/C0SW.java:1606-1610`) |
| Broken frame/noise/TLV violations | `C28951Rz` | `ConnectionThread/connect/socket/disconnect/noise` + `c1fb.A0C()` | from `C51075Mse.reason` (4/5) (`X/C0SW.java:1437-1442`; `A0S`-catch `X/C0SW.java:251-255`) |
| The goaway marker `GOA` instead of the frame length | `C28931Rx` | `ConnectionThread/connect/socket/goaway` | 6 (`X/C0SW.java:1586-1592`) |
| authkey cannot be read from storage | `C28941Ry` | `ConnectionThread/connect/socket/disconnect/authKey` | 8 (`X/C0SW.java:1593-1600`) |
| Socket IO disconnect | `IOException` | `ConnectionThread/connect/socket/disconnect/io` | — (attempt retry) (`X/C0SW.java:1574-1578`) |

`c1fb.A0C()` (`X/C1FB.java:576-593`) — handling of a "suspected handshake error":
in PQ mode ENABLE it marks the attempt for retry (`Mark for retry`), in ENFORCE —
`PQ fallback blocked`. After disconnect/noise the client
forgets the server static, and the next attempt goes as full XX (the `C1KE.A01`
fork sees `C1KC == null`).

Any `C28861Rq` from these branches reaches the common handler
`ConnectionThread/connect/login/failure type:<t> code:<c>` (`X/C0SW.java:1326-1331`),
the callback `Bpt(e)`, saving into `c1fb.A02`, the report `A16(c1fb, c1sc)`
(`X/C0SW.java:519-543`).

## 5. The outgoing XML queue starts only after `A0x`

Until `<success>` is parsed, no outgoing nodes are sent: the queue
(`push`, `presence`, `encrypt`, `abt` etc.) starts in `A0v` only after
the return from `A0x` — the node reader `C1OV` holds the socket exclusively for
the login sequence. Confirmed by the first-login capture (all lines
after 02:48:52.424 — success): .431 `<iq push>` config (id 1, response 404
item-not-found — the token does not exist yet), .461 `<ib><unified_session id='258531227'/>`,
.462 `<presence type='available'/>`, .636 `<iq xmlns='encrypt' type='set'>`
(prekeys, id 2), .716 `<iq xmlns='abt'><props protocol='1' hash=''/>`. The
server in turn, right after success, sends its post-login nodes: .706
`<ib><edge_routing><routing_info>` (the blob for the next `ED` preamble), .714
a batch of `<notice>`, .720 `<dirty type='groups'>` → `account_sync`, .736
`<ib><gpia><request nonce>`, .745 `<ib><safetynet><integrity nonce>`. These
nodes are no longer part of the `A0y` login sequence — the parsing of
`success`/`failure` ends with a `return` on the very first matching node
(`X/C0SW.java:1073`).

## 6. Summary: login error types (the `C28861Rq.type` field)

| type | Source | Error class |
|------|----------|--------------|
| 0 | `<failure>` with an unknown reason | server |
| 2 | 402 without appeal context | server |
| 3 | 405 (client too old) | server |
| 4 | 500–599 (temporary) | server |
| 5 | 406 (new login) | server |
| 6 | goaway | client |
| 7 | 403 (policy block) | server |
| 8 | authkey unreadable | client |
| 10 | invalid certificate | client |
| 11 | 416 (violation, source_acct) | server |
| 13/14/15 | 402 + appeal_token + code 106/107/109 | server |
