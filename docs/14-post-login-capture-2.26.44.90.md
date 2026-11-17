# Post-login traffic of 2.26.44.90

The traffic of the first XMPP login right after registering a new account on
**2.26.44.90 (264409011)**, device SM-A325F, Android 13.

The session contains 22 distinct `xmlns` families, 812 one-time prekeys in a single upload,
21 `w:stats` batches and 36 `w:mex` queries in the first two minutes. The per-parameter
analysis of every node is in
[15-post-login-chain-2.26.44.90.md](15-post-login-chain-2.26.44.90.md).

## 1. Order of the nodes

| # | Direction | Node | Key data |
|---|---|---|---|
| 1 | in | `<success>` | handshake completed |
| 2 | out | `iq` | `to=s.whatsapp.net`, `xmlns=urn:xmpp:whatsapp:push`, `type=get`, `id=1`, child `config version="1"` |
| 3 | out | `ib` → `unified_session` | `id=127334115` |
| 4 | out | `iq` | `xmlns=w:m`, `type=set`, `id=01`, child `media_conn last_id="49829133"` |
| 5 | out | `iq` → `sign_credential` | ACS v1: `version=1`, `blinded_credential` 32 B |
| 6 | out | `iq` → `tos` | `request/notice id=20210210` |
| 7 | out | `iq` → `encrypt` | prekeys: `identity` 32 B, `registration` 4 B, `type=05`, `list` (812 keys), `skey` + 64-byte signature |
| 8 | out | `ib` → `keystore_attestation` | 3420 B DER |
| 9 | in/out | `ib` → `gpia` | `request nonce` → `jws` |
| 10 | out | `config` (`push`) | `num_acc=2`, `platform=gcm` |
| 11 | out | `ib` → `integrity_payload` | 768 B blob (1024 base64) |
| 12 | out | `w:b` → `lists` | broadcast lists |
| 13 | out | `newsletter` → `my_addons` | `limit=1000`, and the status variant `type=status limit=100` |
| 14 | out | `get_user_disclosures` | `t=1794915236` |
| 15 | out | `usync` | contact sync, `context=registration`, `addressing_mode=lid` |
| 16 | out | `urn:xmpp:whatsapp:account` | `crypto action=create`, 32-byte `google` key |
| 17 | out | `presence` | `type=available` |
| 18 | out | `sync_results` | `eval_ts_sec=1794915238` |
| 19 | out | `iq` `w:stats` | `add t=…` + encrypted WAMSys blob (830–1278 B) |
| 20 | out | `w:comms:chat` | `query/blocking_status` |
| 21 | out | `w:comms` → `event` | `promotion_id`, `timestamp_sec`, `logdata`, `type=impression` |
| 22 | out | `ent:silent_nonce` | 16-byte nonce (inbound notification type, answered by an ack) |
| 23 | out | `sign_credential` | ACS v2: `version=2`, `project_name=WA_StatusMusic` |
| 24 | out | `w:mex` | `trace/flow_id` + `query query_id` (JSON body) |

Nodes are emitted from several threads, so the wire order may interleave; the table is the
order in which the client assembled them.

## 2. What happens after `<success>` — summary

1. **Push/config bootstrap**: push config request, `unified_session`, media connection.
2. **Key material**: the prekey upload with the identity key, the signed prekey and 812
   one-time prekeys.
3. **Attestation**: one `keystore_attestation` blob (3420 bytes) with a 32-byte caller
   challenge; sent right after `unified_session`, before the first stats batch.
4. **Integrity**: `integrity_payload` (768 bytes of encrypted JSON) in the same burst;
   `gpia/jws` answers the server integrity nonce and carries a Play Integrity JWS.
5. **Account setup**: `num_acc`/`platform`, `tos`, `get_user_disclosures`, `my_addons`,
   `w:b lists`, `usync` contact sync, account crypto and backup tokens.
6. **Sync**: `sync_results` and the `w:stats` batches run in parallel with the bootstrap.
7. **Comms**: the promotions/telemetry channel (`w:comms:chat`, `event`, `optoutlist`).
8. **ACS**: v1 immediately after login, v2 later in the session.
9. **Silent nonce**: `ent:silent_nonce` is an inbound notification; the client answers with
   an ack.

## 3. Related documents

- [13-delta-2.26.44.90.md](13-delta-2.26.44.90.md) — delta versus 2.26.35.75.
- [15-post-login-chain-2.26.44.90.md](15-post-login-chain-2.26.44.90.md) — per-parameter builders.
- [12-post-login/keystore-attestation-2.26.44.90.md](12-post-login/keystore-attestation-2.26.44.90.md) — attestation details.
