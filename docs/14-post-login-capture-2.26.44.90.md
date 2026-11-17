# Post-login traffic of 2.26.44.90 (live capture)

Live capture of the first XMPP login right after registering a new account on
**2.26.44.90 (264409011)**, device SM-A325F, Android 13.

Raw log: `hooks/wa_agent_log_registration.txt` (5806 lines).

## 1. How it was captured

Frida 17 agent compiled with `frida-java-bridge` (`hooks/agent/agent.ts`) and loaded
through `frida-compile`; plain `Java.use` in a Python `create_script` does not work on
Frida 17 because the bridges are no longer bundled with the runtime
([frida docs](https://frida.re/docs/bridges/)).

Hooked methods:

| Target | Runtime name | What it yields |
|---|---|---|
| decoder | `X.14q.A08()` | incoming node tags |
| encoder | `X.1sR.A05(OutputStream, String, boolean, boolean)` | every string written: tags, attribute keys and values |
| encoder | `X.1sR.A06(OutputStream, byte[], boolean)` | binary payload sizes and hex heads |
| payload provider | `X.1oJ.A00(X.1oI)` | ClientPayload assembly events |
| payload bytes | `X.1oO.toByteArray()` | full ClientPayload bytes for offline protobuf parsing |
| attestation | `X.17x.A04(X.17x, Integer, byte[])` | challenge `p2` and chain length |

Note: reading node fields through reflection (`getDeclaredFields`, `A0Q()`, `A0R()`) makes
WhatsApp crash with SIGSEGV; hooking the writers `A05`/`A06` avoids touching node objects
and is stable.

## 2. Timeline

`W<n>` = global write sequence number in the capture; sizes are bytes of the following
binary write.

| # | Dir | Node | Key data |
|---|---|---|---|
| — | in | `<success>` | handshake completed |
| W1 | out | `iq` | `to=s.whatsapp.net`, `xmlns=urn:xmpp:whatsapp:push`, `type=get`, `id=1`, child `config version="1"` |
| W13 | out | `ib` → `unified_session` | `id=127334115` |
| W88 | out | `iq` | `xmlns=w:m`, `type=set`, `id=01`, child `media_conn last_id="49829133"` |
| W98 | out | `iq` → `sign_credential` | ACS v1: `version=1`, `blinded_credential` 32 B |
| W118 | out | `iq` → `encrypt` | prekey upload: `identity` 32 B, `registration`=regid, `type`, `skey`, `list` |
| W4203 | out | `ib` → `keystore_attestation` | **3420 B DER** (single blob, no attributes) |
| W4235 | out | `num_acc` | account number field |
| W4242 | out | `ib` → `integrity_payload` | **1024 B base64** = 768 B blob (`IV[16] ‖ AES-CBC`) |
| W4287 | out | `ib` → `gpia` → `jws` | response to the server integrity nonce |
| W4447 | out | `w:b` → `lists` | business lists |
| W4479 | out | `newsletter` | newsletter metadata |
| W4485 | out | `my_addons` | addon list |
| W4567 | out | `tos` | terms-of-service version |
| W4571 | out | `get_user_disclosures` | disclosure query |
| W4764 | out | `presence` | `type=available` |
| W4963 | out | `sync_results` | initial sync result |
| W5090 | out | `iq` `xmlns=w:stats` | `add t=…` + encrypted stats blob |
| W5104 | out | `w:comms` → `event` | `promotion_id`, `timestamp_sec` |
| W5152 | out | `sign_credential` | ACS v2: `version=2`, `project_name`, `config_id` |
| W5360 | out | `ent:silent_nonce` | 16 B nonce |
| — | — | `w:mex` / `trace` / `query_id` | GraphQL-style requests, repeated ~35 times |

Unique node/attribute names observed (heuristic extraction, 80 raw tokens):

```
addressing_mode, allow_mutation, config, encrypt, ent:silent_nonce, event,
get_user_disclosures, gpia, ib, identity, impression, integrity_payload, iq, jws,
keystore_attestation, limit, lists, logdata, media_conn, missing, my_addons,
newsletter, num_acc, optoutlist, presence, promotion_id, query_id, registration,
sid, sign_credential, sync_results, timestamp_sec, tos, trace, unified_session,
w:b, w:comms, w:mex, w:stats
```

## 3. What happens after `<success>` — observations

1. **Push/config bootstrap**: the client immediately asks for push config (`iq` with
   `urn:xmpp:whatsapp:push`) and reports `unified_session`, then registers the media
   connection (`w:m media_conn`).
2. **Key material**: the prekey upload (`<iq xmlns="encrypt">`) contains the identity key,
   the signed prekey (`skey`), the registration id and the prekey list — it is sent within
   the first seconds, before most other stanzas.
3. **Attestation**: `keystore_attestation` is sent once, as a single binary blob of
   3420 bytes; the challenge random part `p2` (32 bytes) comes from the caller, not from
   `SecureRandom`.
4. **Integrity**: `integrity_payload` is 768 bytes of encrypted JSON (1024 base64 chars);
   it is sent in the same burst. `gpia`/`jws` is a *response* to the server's
   `<ib><gpia><request nonce=…>` and carries a Play Integrity JWS.
5. **Account setup**: `num_acc`, `tos`, `get_user_disclosures`, `my_addons`,
   `newsletter`, `w:b lists` — these are the account/business bootstrap queries that were
   not present in the 2.26.35.75 documentation.
6. **Sync**: `sync_results` and the `w:stats` batches (encrypted blobs, 800–1200 B) run in
   parallel with the bootstrap.
7. **Comms/marketing**: `w:comms event` with `promotion_id` and `timestamp_sec`,
   `logdata`, `impression`, `optoutlist` — a promotions/telemetry channel that is new to
   the documentation.
8. **ACS**: two variants are visible — v1 (`sign_credential version=1`,
   `blinded_credential` 32 B) right after login, and v2 (`version=2`, `project_name`,
   `config_id`) later in the session, matching `acs-privatestats.md`.
9. **Silent nonce**: `ent:silent_nonce` (16 B) is exchanged server→client, presumably for
   ent-tier verification.

## 4. Reproduce

```bash
# on the PC
python hooks/agent/run_agent.py         # spawns / re-attaches, writes hooks/wa_agent_log.txt
```

Then register or log in on the device; the log is appended.

Companion documents:

- [13-delta-2.26.44.90.md](13-delta-2.26.44.90.md) — full delta versus 2.26.35.75.
- [12-post-login/keystore-attestation-2.26.44.90.md](12-post-login/keystore-attestation-2.26.44.90.md) — attestation details.
