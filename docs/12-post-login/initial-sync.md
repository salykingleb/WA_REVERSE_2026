# Initial synchronization after success: all the remaining nodes

WhatsApp 2.26.35.75 (263507522). Wire: live capture of the first login, account `6615@s.whatsapp.net` / `4312@lid`. Listed here are the nodes of the first second and the first 50 seconds that are NOT device attestation (attestations are in separate files of the chapter) and not a prekey upload.

## Push config: get → 404 → set

| Time | Direction | Node | Details |
|---|---|---|---|
| 02:48:52.431 | client | `<iq xmlns='urn:xmpp:whatsapp:push' type='get' id='1'><config version='1'/></iq>` | request the current push config |
| 02:48:52.884 | server | `<iq type='error' id='1'><error code='404' text='item-not-found'/>` | there is no token on the server — a freshly created account |
| 02:48:52.907 | client | `<iq xmlns='urn:xmpp:whatsapp:push' type='set' id='0c'><config id='fHSuGq_MTpSjBfJDKE75uU:APA91bHxN7GkCgjBu_kwymRvBSXx9…` | FCM token registration |
| 02:48:53.750 | server | `<iq type='result' id='0c'/>` | token accepted |

The "get, then set" scheme is visible only on the first login: afterwards the token already sits on the server.

## unified_session and presence

- 02:48:52.461 and again at 02:48:52.497 — `<ib><unified_session id='258531227'/>`: binding of the socket to the unified-session identifier (sent twice, flags=1 encoding).
- 02:48:52.462 — `<presence type='available'/>`: the session declares itself available, NOT passive. Passive mode is enabled later by a separate IQ: 02:49:32.986 `<iq id='4' xmlns='passive' type='set'>`, the response `<active/>` at 02:49:36.837.

## edge_routing

02:48:52.706 server: `<ib from='s.whatsapp.net'><edge_routing><routing_info>
</routing_info></edge_routing></ib>` (the blob is binary; the hook prints it empty). `X/C27871Nj` saves the blob to the prefs `routing_info`. On the next connect the client will send it BEFORE the Noise preamble: the edge header `ED\x00\x01` (`C1KE.A0A`), 3 big-endian length bytes, the blob itself (base64-decode with flag 3) — routing to the right edge of the `location` datacenter (`vll` here) without an extra resolve.

## Legal notices

02:48:52.714 the server sends a single batch of `<notice …/>` with stage 0, `t='1788392932'` (the login second), type 2:

```text
id=20250922 version=2   id=20601218 version=10   id=20601229 version=4
id=20601230 version=4   id=20610203 version=3    id=20610204 version=3
id=20610273 version=3   id=20610208 version=1    id=20610209 version=2
id=20610230 version=3   id=20610231 …
```

Later (02:49:21.678/702) the client requests details: `<iq xmlns='tos' id='01c'> <get_user_disclosures t='1788392961'/>` and `id='01d'` `<get_disclosure_stage_by_id id='20601230' …/>` — the responses come back as the same batch of notices. These are disclosures of legal texts; stage 0 is the initial display stage.

## abt: AB props

- 02:48:52.716 client: `<iq type='get' id='07' xmlns='abt'><props protocol='1' hash=''/></iq>`. An empty `hash` — there is no local set of props (first login). `abprops='185'` from `<success>` is the NUMBER of the set on the server, not the props themselves.
- 02:48:54.214 server: `<iq type='result' id='07'><props protocol='1' ab_key='fnF,KNV,1K3,Rv,L2,3TG,St,5Wa,3M,6F,I6,Lf,N,D3,3d,Fp,5W,7w,83,9W,17,3j,5g,10,5t,9C,BN,I,S,5H,1x,1i,6,42,1E,8,3B,3G,1b,I,w,2Q,…'>`. Afterwards all the session's AB flags (1934, 2076, 12964, 14916, 16017, etc.) are read from this set (`C05D.A0w`).

## dirty: mandatory re-synchronization

- 02:48:52.720 `<ib><dirty type='groups' timestamp='1788392932'/>`
- 02:48:52.728 `<ib><dirty type='account_sync' timestamp='1788392932'/>`

Both marks require the client to re-synchronize the corresponding data. In practice, groups are fetched later via mex (`xwa2_group_query_participating_groups`, result `{"data":{…":[]}}` at 02:49:22.197), account_sync via usync and profile GraphQL queries.

## ToS: terms acceptance

- 02:48:52.756 client: `<iq id='0b' xmlns='tos' type='get'><request><notice id='20210210'/></request></iq>`
- 02:48:53.719 server: `<iq type='result' id='0b'><tos refresh='86400'><notice id='20210210' state='true'/></tos></iq>` — the terms are accepted, `refresh=86400` (a day) until the next check.

## media_conn

- 02:48:52.739 client: `<iq xmlns='w:m' type='set' id='0a'><media_conn/>`
- 02:48:53.715 server: `auth` 56 bytes, `ttl='300'`, `auth_ttl='21600'`, `max_buckets='12'`, `id='38669907'`, `is_new='1'`, `ip_token` 51 bytes, `set_ip_token='1'`, a list of `<host hostname='me…'>…` — media hosts for content upload/download.

## usync: contacts after registration

- 02:49:21.867 client: `<iq xmlns='usync' id='01f' type='get'><usync sid='ContactSyncHelperKt/sync_sid_multi_iq_9fa106f6-dfb7-4497-83c0-49a47a040a50' index='0' last='true' mode='full' context='registration'><query><contact addressing_mode='lid'/><status/><business><verified_name/><profile v='16372'/></business><devices version='2'/><disappearing_mode/><username/><text_status/></query>
  <list><user><contact>10 bytes</contact>…`
- 02:49:23.231 server: `<iq from='4312@lid' type='result' id='01f'>` — the result over LID addressing.

`context='registration'` — this usync exists only because the login is the first one: the phone book is checked immediately after account creation. Next, the client requests encryption sessions for counterparts: 02:49:24.275 `<iq id='023' xmlns='encrypt' type='get'><identity><user jid='0287:0@lid'/>…` → 02:49:24.695 the response with the list `<user jid='4315@lid'><type>\x05</type><identity>…`.

## The backup key: `<crypto action='create'><google>`

- 02:49:21.715 client: `<iq xmlns='urn:xmpp:whatsapp:account' type='get' id='01e'><crypto action='create'><google>
  pcm41mum7wpl88Q8npPf/P51fDAAcHWsRQ/RO1TgFJI=</google></crypto></iq>` — 32 bytes of client material.
- 02:49:22.214 server: `<iq type='result' id='01e'><crypto version='2'><code>
  [32 bytes of salt]</code><password>6wC15/eOXNHHGOaZbF5hcsdqcBdlW1Ac3zw2dN8MfUo=
  </password></crypto></iq>`.

This is `BackupSendMethods.A01`/`A03`, stanza 74, log `createCipherKey` (in jadx: `…, A01(strA0F, bArr), strA0F, 74, 32000L`). A successful response must contain `version`, `code` (the server salt) and `password`; if any of them is missing — the `C28891Rt` exception with the text `missing version node` / `missing serverSalt node` / `missing password node`. Re-reading an already created key is a different IQ, `action='get'` (with the `google` + `code` nodes). This is NOT device attestation: the 32 bytes in `<google>` are the client's contribution to the Google backup encryption key.

## The recovery nonce: two channels (use_case 11)

### `<ib><recovery_nonce>` → message 289 → `X/C20650wB`

02:49:42.625 server: `<ib from='s.whatsapp.net'><recovery_nonce code='64 bytes' use_case='2 bytes' request_id='36 bytes'/>`. The handler requires the `code` and `use_case` attributes, and `use_case` must parse as an int (`C0C5.A07`, base 10). Value 11 goes into background processing. 547 is an empty handler for the advertising nonce (`throw new NullPointerException("handleNonceNotification")` — nothing is attached). The remaining numbers are silently consumed.

### `<notification type='ent:silent_nonce'>`

02:49:42.661 server:

```xml
<notification from='s.whatsapp.net' type='ent:silent_nonce' id='2327453894' t='1788392982'>
 <use_case>11</use_case>
 <account_recovery_nonce>tNMFtqsSAf78zKrWvKzCZESTZAfvV2CsvXTVG39StvlFUJjYQn3BD8AoyZT91FKr</account_recovery_nonce>
 <request_id>b7983fa0-22ae-4409-8f16-d71e78fce944</request_id>
</notification>
```

(`account_recovery_nonce` — 76 b64url chars = 57 bytes.) The type is registered in `X/C7G` (line 142): `A00("ACCOUNT_RECOVERY_SILENT_NONCE", "ent:silent_nonce", 42)`, message code **276**. `X/C20640wA.A07` reads the `account_recovery_nonce` and `use_case` children ONLY when all the conditions hold:

```java
if (i != 276 || !((C00D)…).A0w(14916) ||
    (c0p3A0F = c0p3.A0F("account_recovery_nonce")) == null || …A0I() == null ||
    (c0p3A0F2 = c0p3.A0F("use_case")) == null || …A0I() == null) { /* авто-ack */ }
```

otherwise the notification ends with an automatic ack. In the capture the ack went out immediately: 02:49:42.682 `<ack id='2327453894' to='s.whatsapp.net' class='notification' type='ent:silent_nonce'/>` — after 21 ms. This is a delivery acknowledgment, not an exchange.

`AccountRecoveryManager.A00` (`processDeferredNonce`) then skips the nonce if AB **16017** is enabled:

```text
AccountRecoveryManager/processDeferredNonce: encryption enabled, skipping (Stage 2 required)
```

or if credentials already exist: `valid credentials already exist, skipping`; otherwise `processing deferred nonce for useCase=…`. The `exchangeNonce` GraphQL exchange never fully appears in this capture (only neighboring mex requests with `nonce_encryption_key` — the Stage 2 PEM key — are visible, starting at 02:49:40.630).

## Other traffic of the same minute (for completeness)

| Time | Node | Meaning |
|---|---|---|
| 02:48:52.509 / .925 | mex id `06` `{"input":"PASSKEYS"}` | account passkeys request (argo) |
| 02:48:52.700 / 02:48:53.693 | `w:stats` id `08` | WAM metrics, `t='1788392932'` |
| 02:48:53.699 | `w:stats` id `0d` | the second batch of metrics |
| 02:48:55.668 / 02:48:56.360 | `<iq id='3' xmlns='status' type='set'><status> </status>` | setting the new account's empty text status |
| 02:49:21.511–.581 | `w:b`, `newsletter`, mex | a snapshot of the new account's lists/channels/profiles |
| 02:49:21.599 / 02:49:22.232 | `w:profile:picture` target=self → 404 | there is no avatar yet |
| 02:49:24.381+ | a batch of `w:profile:picture` over the contacts | counterpart avatar previews |
| 02:49:37.225–.301 | `<ib><offline_preview …/>`, `<notification type='psa'>` | server promo notifications for the new account |

## Where to look in the code

- Push: the `urn:xmpp:whatsapp:push` handlers in `X/C27871Nj.java`.
- edge_routing → the preamble: `X/C1KE.java` `A0A`, prefs `routing_info`.
- ToS/disclosures: the `xmlns='tos'` listeners, `get_user_disclosures`.
- The backup key: `com/whatsapp/infra/backup/encryption/BackupSendMethods.java` (`A01`/`A03`, stanza 74), errors `X/C28891Rt.java`.
- Recovery: `X/C20650wB.java` (message 289), `X/C20640wA.java` `A07` (276, AB 14916), `X/C7G.java` (42), `com/whatsapp/fbusers/recovery/AccountRecoveryManager.java` `A00` (AB 16017, exchangeNonce).
- usync: `ContactSyncHelperKt/sync_sid_multi_iq_*`, the query profile v='16372'.
