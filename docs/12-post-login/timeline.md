# Timeline of the first second after `<success>`

WhatsApp build 2.26.35.75 (versionCode 263507522). Wire: a live capture of the first login (phone SM-A325F, account `6615@s.whatsapp.net` / `4312@lid`, first login after registration). Java analysis: jadx decompilation; native: a dump of the loaded `libs.spo/libwhatsapp.so` (base `0x7cfd633000`, code `0x7cfdacf000`).

## What `<success>` is and who parses it

`<success>` is the server's response to the ClientFinish of the full Noise handshake (`WA\x06\x03`, Noise_XX_25519_AESGCM_SHA256). It is parsed by `X/C0SW.A0y`:

| Attribute | Live value of this session | What the client does |
|---|---|---|
| `t` | `1788392932` = 2026-09-02 23:48:52 UTC | server time, in seconds; the client memorizes it together with the local seconds to compute the clock offset. A non-number — `invalid server time` |
| `location` | `vll` | datacenter code, written to prefs if shorter than 40 characters (`A0q`) |
| `pn` | `6615@s.whatsapp.net` | the phone JID, goes into `C42571uZ` |
| `lid` | `4312@lid` | LID; on the first login `lid-lifecycle/login-success` is written |
| `abprops` | `185` | the number of the AB-props set (not the props themselves) |
| `creation` | `1788392926` | account creation timestamp; `A0y` does not read it |

`t` and `creation` differ by 6 seconds: the account was created at 1788392926, the socket was accepted at 1788392932. Until `<success>` has been parsed, the outgoing XML queue (`push`, `presence`, `encrypt`, `abt`) does not start — it is launched in `A0v` only after the return from `A0x`.

Important for hooks: from this moment on, `t` from success appears as the challenge in the keystore attestation (the bytes `6a 98 b5 e4` in the 8-byte challenge prefix) and as `t=` in the server's `<notice>`/`<dirty>`.

## Chronology of the first second (live timestamps 02:48:52.xxx)

| Time | Who | Node | Meaning / client code |
|---|---|---|---|
| .424 | server | `<success t='1788392932' …/>` | parsed by `C0SW.A0y`; the entire table below is already after it |
| .431 | client | `<iq xmlns='urn:xmpp:whatsapp:push' type='get' id='1'><config version='1'/>` | push config request; the 404 `item-not-found` response at .884 — the server has no FCM token yet |
| .461 | client | `<ib><unified_session id='258531227'/></ib>` | unified-session binding; sent twice (repeat at .497) |
| .462 | client | `<presence type='available'/>` | the session is not a passive session |
| .509 | client | `<iq xmlns='w:mex' id='06'>` `{"queryId":"25435755019399064","variables":{"input":"PASSKEYS"}}` | passkeys GraphQL query; the `argo` result at 02:48:52.925 |
| .636 | client | `<iq id='2' xmlns='encrypt' type='set'>` | prekey upload: identity 32B + registration 4B + type `0x05` + 812 one-time keys + skey; result for id 2 at 02:48:54.220 |
| .700 | client | `<iq xmlns='w:stats' type='set' id='08'><add t='1788392932'>` | WAM metrics, `t` = the login second; result at 02:48:53.693 |
| .706 | server | `<ib><edge_routing><routing_info>` | `C27871Nj` saves the blob; on the next connect it will be sent as the `ED\x00\x01` preamble |
| .714 | server | a batch of `<notice>` | legal notices stage 0 (see the initial synchronization file) |
| .716 | client | `<iq xmlns='abt' id='07'><props protocol='1' hash=''/>` | empty hash — no local props; `abprops=185` from success is the set number; the result with `ab_key='fnF,KNV,…'` at 02:48:54.214 |
| .720 | server | `<ib><dirty type='groups' timestamp='1788392932'/>` | mark of mandatory group re-synchronization |
| .728 | server | `<ib><dirty type='account_sync' timestamp='1788392932'/>` | the second re-synchronization mark |
| .729 | client | `<iq xmlns='privatestats' type='get' id='09'>` v1 | blinded_credential of exactly 32 bytes; assembled by the native `JniBridge` case 7 |
| .736 | server | `<ib><gpia><request nonce='AeGnsZKe2wR1JPjPnMSTdyTWCM0Kb-aP…'>` | nonce 57 bytes (76 b64url chars); `C27871Nj` → `C1HI.A1J` |
| .739 | client | `<iq xmlns='w:m' type='set' id='0a'><media_conn/>` | media hosts request (already synchronization, not verification) |
| .745 | server | `<ib><safetynet><integrity nonce='ATawnSKrOzJC7B6lySW4LiPRrfNIcHONk…'>` | nonce 66 bytes (88 chars); `C27871Nj` → `C1HI.A1I` |
| .756 | client | `<iq xmlns='tos' id='0b'><request><notice id='20210210'/></request>` | ToS acceptance; the result `state='true'` at 02:48:53.719 |
| .769 | client | `<ib><keystore_attestation>` | a chain of 4 DER, 3417 bytes, 24 ms after the safetynet nonce (.745) |
| .780 | client | `<ib><integrity_payload>` | 768 bytes (48 AES blocks), 35 ms after the safetynet nonce; not a JWS |
| .907 | client | `<iq xmlns='urn:xmpp:whatsapp:push' type='set' id='0c'>` | FCM token registration `fHSuGq_MTpSjBfJDKE75uU:APA91b…`; result at 02:48:53.750 |

## What arrives later in the same session

| Time | Who | Node | Meaning |
|---|---|---|---|
| 02:48:53.726 | server | result id `09` | ACS v1 signature: signed_credential 32B + acs_public_key 32B + dleq_proof c,s of 32B each + config_id empty |
| 02:48:54.220 | server | result id `2` | acknowledgment of prekey receipt |
| 02:48:57.266 | client | `<ib><gpia><jws>` | 2976 bytes of binary token; **4.53 s** after the nonce at .736 — the only attestation step that waited for the external network (Google Play) |
| 02:49:21.715 | client | `<iq xmlns='urn:xmpp:whatsapp:account' id='01e'><crypto action='create'><google>` | 32 bytes of client backup key material; result at 02:49:22.214 |
| 02:49:21.867 | client | `<iq xmlns='usync' id='01f'>` | contact synchronization, `context='registration'` |
| 02:49:32.986 | client | `<iq id='4' xmlns='passive' type='set'>` | switch to passive; the result `<active/>` at 02:49:36.837 |
| 02:49:40.800 | client | privatestats version 2 `WA_StatusMusic` | the second ACS request, this time from Java; result at 02:49:41.464 |
| 02:49:42.625 | server | `<ib><recovery_nonce code='64 bytes' use_case='2 bytes' request_id='36 bytes'/>` | message 289, `X/C20650wB` |
| 02:49:42.661 | server | `<notification type='ent:silent_nonce' id='2327453894'>` | use_case 11; client ack at 02:49:42.682 |

## Login verification versus ordinary synchronization

The server initiates the verifications itself — the client did not send them in the handshake. Division by meaning:

**Integrity/attestation checks (the server triggers them, the client must respond):**

1. `<gpia><request nonce>` (.736) — Play Integrity via Google Play; the `<jws>` response is the only step with external network waiting (4.53 s).
2. `<safetynet><integrity nonce>` (.745) — local attestation; within a single 24–35 ms window the client hands over TWO nodes: `<ib><keystore_attestation>` (.769) and `<ib><integrity_payload>` (.780). Both are assembled by the native code within one periodic request (`DTA` case 10, `jvidispatchIOOOO` id 5).
3. privatestats v1 (.729) — an ACS blind signature, initiated by the client itself (native); the server answers with the signature at 02:48:53.726; the signature is carried back into the native code via `jvidispatchIOOOO` id 4.

**Account-key operations of the first login (without them the account is not functional):**

4. `<iq xmlns='encrypt'>` (.636) — the prekey pool upload (see the separate file).
5. `<crypto action='create'><google>` (02:49:21) — the backup key, not device attestation.

**Ordinary initial synchronization (would also happen on a repeat login):**

6. push config get/set, unified_session, presence available, mex PASSKEYS, w:stats, edge_routing, the notice batch, abt, dirty groups/account_sync, tos, media_conn, the contacts usync, the passive switch, recovery_nonce / ent:silent_nonce (use_case 11).

## Observations on the acknowledgment mechanism

- The server acknowledges IQ requests (`type='result'` for every id: 1→404, 06, 08, 0a, 0b, 09, 0c, 2, 07 …). The `<ib>` nodes (gpia, safetynet, keystore_attestation, integrity_payload, edge_routing, notice, dirty, recovery_nonce) receive no acknowledgment from the server in this capture — there is no acknowledgment for them in the capture's protocol; the delivery guarantee is provided by the Noise socket framing itself.
- The order .729 → .736 → .745 shows: the native code initiates ACS before receiving the server nonces; gpia and safetynet arrive 9 ms apart.
- No server nonce sits in the client's responses in plaintext ON THE WIRE (the payload is AES-256-CBC encrypted, the jws is a binary JWE); but inside the decrypted integrity_payload the server's safetynet nonce sits in `_n` in plaintext (live confirmation on 2.26.37.73).
- Metrics: the gpia path closes the `GPIA_DURATION` metric (`CYG.A01`), the safetynet path writes `safety-net-attestation` = `success`/`failed` (`c07570Ws`).

## Where to look in the code

- Success parsing: `X/C0SW.java` `A0y` (around line 904), `A0q`.
- The post-success dispatcher of incoming `<ib>`: `X/C27871Nj.java` around lines 270–320 (gpia, safetynet, edge_routing).
- Messages 179/254: `X/C1US.java`, `X/C1HI.java` (`A1I`, `A1J`), `X/DTA.java` case 10, `X/C12G.java` `BD5`.
- Prekeys: see `prekeys-encrypt.md`.
- Attestations: see `keystore-attestation.md`,
  `integrity-payload.md`, `gpia-jws.md`.
- ACS: see `acs-privatestats.md`.
- The rest of the synchronization: see `initial-sync.md`.
