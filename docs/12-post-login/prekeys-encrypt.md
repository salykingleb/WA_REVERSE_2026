# Prekey upload: `<iq xmlns='encrypt' type='set'>` immediately after success

WhatsApp 2.26.35.75 (263507522). Wire: live capture of the first login; the node is sent at 02:48:52.636 — 212 ms after `<success t='1788392932'>` (.424), with id `2`. This is the first meaningful SET of the session: before it, the client had only had time to request the push config (id 1) and send `unified_session`/`presence`.

## Live node (structure)

```xml
<iq id='2' xmlns='encrypt' type='set' to='s.whatsapp.net'>
  <identity>eXHfoDtXkeGBZAO0FNjHFTjYo5NXRsulzxr2C2AAaFQ=</identity>
  <registration>J/hw3g==</registration>
  <type>\x05</type>
  <list>
    <key><id>uBZd</id><value>GWFnIgwP3F/au0+tcKZn7ZAy6NSACWTtvsbHkyIOmy4=</value></key>
    … ещё 811 ключей с последовательными id …
  </list>
  <skey>
    <id>bYvi</id>
    <value>20796bzw94Yeb9A2Z7od9KEan9e5iQqZ2V3uADzVfEI=</value>
    <signature>0LRqyRGFFqiNAUPkyIQW/ZyCmvxnE8WrT1qV7xxLmNG…+f4lYazKkxCtmhxiZCjRwZJsofBfmJppCw==</signature>
  </skey>
</iq>
```

Verified sizes of this session:

| Element | Live value | Size |
|---|---|---|
| `<identity>` | `eXHfoDtX…2AAaFQ=` | 32 bytes (44 base64 chars) |
| `<registration>` | `J/hw3g==` → `27 f8 70 de` | 4 bytes BE = **670593246** |
| `<type>` | byte `0x05` | 1 byte = Curve25519 (ECC type 5) |
| `<list>` | 812 one-time prekeys | each: id 3B + value 32B |
| first OTP id | `uBZd` → `b8 16 5d` = 12064349 | consecutive 3-byte BE increments |
| `<skey>` id | `bYvi` → `6d 8b e2` = 7179234 | 3 bytes |
| `<skey>` value | `20796bzw…` | 32 bytes |
| `<skey>` signature | `0LRqyRGF…` | 64 bytes (88 b64 chars) — an Ed25519 signature made with the identity key |

The server's response: `<iq type='result' id='2'/>` at 02:48:54.220 — an empty result, the upload is accepted. The second time the `xmlns='encrypt'` node appears in this session is only as a GET (id `023`, 02:49:24.275) — a request for the identities of counterparts to set up sessions (`<identity><user jid='0287:0@lid'/>…`); that is already a consequence of the contacts usync, not an upload.

## Correspondence to registration parameters

The key material is the same that the installation submitted during HTTP registration at `/v2/register` (live parameters of the same schema: `e_ident`, `e_regid`, `e_keytype`, `e_skey_id`, `e_skey_val`, `e_skey_sig`):

| /v2/register parameter | encrypt IQ node | Live correspondence |
|---|---|---|
| `e_ident` (32B b64) | `<identity>` | the installation's public static X25519 identity key |
| `e_regid` (4B) | `<registration>` | registration id; here `0x27f870de` = 670593246 |
| `e_keytype` = `BQ` (b64 of byte `0x05`) | `<type>` | the same byte 0x05 |
| `e_skey_id` (3B BE) | `skey/id` | signed prekey id |
| `e_skey_val` (32B) | `skey/value` | the public signed prekey |
| `e_skey_sig` (64B) | `skey/signature` | signature by the identity key |
| — (not uploaded in a batch at registration) | `list/key` × 812 | the one-time prekey pool, generated for the first login |

In other words, registration gives the server the static triple identity + registration + signed prekey, while the first XMPP login adds the pool of one-time keys with which the server will hand out sessions to incoming counterparts (Sender Key / session setup). Verification on a live example of another installation (live capture): `e_regid=extZ9w → 0x7b1b59f7 = 2065390071 ≤ 2147483646`, `e_skey_id=E-mO → 1304974 (3B BE)`, `e_keytype=BQ → 0x05` — the formats match the parsing of this session's node.

## Why these keys were not in the handshake

The ClientPayload (`X/C1HN`, DI set 6929) on an ordinary phone with a JID does not carry prekeys: the `X.1JU` block with `e_ident` / `e_skey_*` / `build_hash` is executed only when `UserJid == null`. That is companion (linked device) registration: the companion has no phone JID at the moment of connection, so its keys are sent inside the encrypted `ClientFinish.payload` of the handshake. The phone has a JID (`username` protobuf field 1), and its keys are accepted by the server only after `<success>` — as a separate `<iq xmlns='encrypt'>` with id `2`.

The order is confirmed by behaviors in the code:

- `X/C1HS.java` — protobuf ClientPayload; the `username` field (1) is filled from `X.1JV` only when a JID is present.
- `X.1JU` — device-pairing filler, condition `UserJid == null`.
- The client's static key in ClientFinish (`ClientFinish.static`, `encrypt_cs`) is the installation's authkey, the same one that appeared at `/v2/register`; the server recognizes the account by the pair "username + static".

## Observations on the pool size

812 one-time keys in a single SET is the initial upload of a fresh account (the server does not have a single OTP for it yet). The identifiers run consecutively from `0xb8165d`; each `<value>` is exactly 32 bytes of X25519. The skey signature (64 bytes) lets the server verify the binding of the signed prekey to the identity without a separate certificate. Further OTP top-ups go over the same xmlns as the pool is consumed (there is no repeat upload in this capture — the account has just been created).

## Where to look in the code

- IQ assembly and sending: handlers in `X/C1US.java` (the post-success queue), the node builder — the cryptographic key-sending service (the `SendKeysIq` path in the jadx tree `com/whatsapp/…/key`).
- ClientPayload and `X.1JU`: `X/C1HN.java`, `X/C1US.java` cases 6556–6561 and 6567.
- The GET branch (id `023`, counterpart identities): the same `xmlns='encrypt'`, `type='get'`; the listener parses `<user jid=…><type>…</type><identity>…`.
- Live wire: the first-login capture, lines with timestamps 02:48:52.636 (SET), 02:48:54.220 (result), 02:49:24.275 (GET), 02:49:24.695 (the response with the `<user>` list).

## Related files of the chapter

- `timeline.md` — the node's place in the first second.
- `initial-sync.md` — the encrypt GET requests after usync.
