# ClientFinish: the client static key and the login payload

Build 2.26.35.75 (263507522). The concluding message of the full handshake — here
the client reveals its static key and login for the first time. Protobuf `X/C1LY.java`;
sending — span `send_client_finish` (`C1KS` code 26, `X/C1KS.java:56`);
state machine state `client_finish` (code 8, `X/C1KW.java:23`).

## 1. ClientFinish protobuf (`X/C1LY.java:15-21`)

| Field | Number | Type | Contents |
|------|-------|-----|------------|
| `static` | 1 (`STATIC_FIELD_NUMBER`) | bytes | encrypted client static public key (X25519, 32 bytes) |
| `payload` | 2 (`PAYLOAD_FIELD_NUMBER`) | bytes | encrypted ClientPayload (protobuf `C1HS`, chapter 11 of the guide) |
| `extendedCiphertext` | 3 (`EXTENDED_CIPHERTEXT_FIELD_NUMBER`) | bytes | ML-KEM encapsulate ciphertext (PQ only) |
| `paddedBytes` | 4 | bytes | padding |
| `simulateXxkemFs` | 5 | bool | test flag |

Frame: `HandshakeMessage { clientFinish }` (field 4 of the `C1LV` wrapper), sent
via `C1KV` — 3 bytes BE of length + protobuf, then `flush` (the "Preamble" chapter).

## 2. What key is in the `static` field

The client's static public key is **the same authkey** that was registered at the
HTTPS endpoint `/v2/register` (object `AnonymousClass139`,
see chapter 09 of the guide: the parameter
`authkey` — 43 base64url characters of the 32-byte pubkey[1:]). Storage —
prefs `keystore`, key `client_static_keypair_enc` (+ `_pwd_enc`),
read as the pair `C1KB {C1Jp clientLoginKeyPair, int A00}` from
`AnonymousClass139.A0C()` (`X/AnonymousClass139.java:417-423`; loading
`C1FB.A0B`, `X/C1FB.java:522-575`). The server recognizes the account by the
pair "username in ClientPayload + this static": the handshake will not accept a
username without the corresponding static, or a static without a username.

The static is sent **encrypted** (`encrypt_cs`, span 10,
`X/C1KS.java:29`): at this point the chain key is already known to both the client
and the server (the ee + es inputs from ServerHello), so the field is encrypted with
the current Noise cipherstream — on the wire this is not a public key.

## 3. The client's order of actions

After the server certificate is accepted (the ServerHello chapter), `C1KE` performs:

| Step | Operation | ClientFinish field | Span (C1KS code) |
|-----|----------|-------------------|-----------------|
| 1 | Take the client static pair `C1Jp` (authkey); encrypt the public key (32 bytes) with the current cipherstream | → `static` (1) | `encrypt_cs` (10) |
| 2 | DH se: the shared secret of client static × server ephemeral; mix into the chain key | — | `ecdh_se` (7) |
| 3 | Encrypt `ClientPayload.toByteArray()` (protobuf `C1HS`) | → `payload` (2) | `encrypt_login_payload` (11) |
| 4 | If there was an ML-KEM key in ServerHello (PQ mode): write the ciphertext saved at the encapsulate step | → `extendedCiphertext` (3) | (the secret is already mixed by span `encapsulate` (9) during ServerHello processing) |
| 5 | Serialize `HandshakeMessage { clientFinish }` and send via `C1KV` | frame | `send_client_finish` (26) |

A subtlety of the order: the encrypted `static` leaves in the frame before/together
with DH se, but the se secret is computed from the same `C1Jp` pair — the server,
knowing the client's ephemeral and static (after decrypting field 1), reproduces the
same input into the key, and decryption of `payload` on its side is only possible
with the correct key. After step 5 the chain key is finalized (`C1KX.A01` — the
HKDF expansion of the final `h` into session keys, `X/C1KX.java:82-99`), the socket
switches to transport encryption (the `C1NM`/`C1NP` wrappers, `X/C1KE.java:161-171`),
and the further stream is already XML nodes (`<success>`/`<failure>`, the next
chapter).

## 4. `extendedCiphertext` with PQ

The field format in ClientFinish is only the encapsulate ciphertext bytes (without
the TLV header; TLV `0x01‖len16‖ct` is the resume-ClientHello format, the ClientHello
chapter). The ML-KEM-512 ciphertext length — an 800-byte pubkey yields a fixed
encapsulation (768 bytes of ciphertext per the ML-KEM-512 specification). The
encapsulation secret has already been mixed into the key during ServerHello
processing (`C1KX.A07`, `X/C1KX.java:202-222`), so there is no repeated mixing at
send time.

## 5. Difference from the resume variant

On resume ClientFinish does not exist: `static` (encrypt_cs), payload
(encrypt_login_payload) and the KEM ciphertext are laid out across the fields of
ClientHello itself (the ClientHello chapter, section 4). Thus `send_client_finish`
(span 26) occurs only in the full route of the state machine
(`client_hello → await_server_hello → handle_server_hello → client_finish →
await_login`), while in the resume route the send span is `send_client_resume` (25).

## 6. What is visible in the live capture

Live capture of the first login (SM-A325F): the FunXmpp hook activates only after
the handshake, so ClientFinish itself is not printed. That the payload was delivered
and accepted is evidenced by the very first event after it — the node
`<success t='1788392932' location='vll' pn='6615@s.whatsapp.net'
lid='4312@lid' abprops='185' creation='1788392926'/>` at 02:48:52.424,
and then `<iq xmlns='encrypt' type='set'>` with prekeys (02:48:52.636) —
the prekeys did not participate in the handshake (chapter 11), which means the login
payload was the only body of ClientFinish.
