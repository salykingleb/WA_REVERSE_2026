# ClientHello: protobuf, state machine, full and resume variants

Build 2.26.35.75 (263507522). The client's first handshake message. Protobuf:
wrapper `X/C1LV.java`, the message itself `X/C1LR.java`. State machine —
`X/C1KW.java`. Hash mixing — `X/C1KX.java`.

## 1. Protobuf schema

`HandshakeMessage` (`X/C1LV.java:14-18`):

| Field | Number | Type | Contents |
|------|-------|-----|------------|
| `clientHello` | 2 (`CLIENT_HELLO_FIELD_NUMBER`) | message `C1LR` | client part |
| `serverHello` | 3 | message `C1LX` | server part (not set in ClientHello) |
| `clientFinish` | 4 | message `C1LY` | (not set in ClientHello) |

The ClientHello frame is `HandshakeMessage { clientHello }`, serialized via
`C1KV` (3 bytes BE of length + body, the "Preamble" chapter).

`ClientHello` (`X/C1LR.java:16-26, 68`):

| Field | Number | Type | When set |
|------|-------|-----|----------------|
| `ephemeral` | 1 (`EPHEMERAL_FIELD_NUMBER`) | bytes | always |
| `static` | 2 | bytes | classic resume only (IK) |
| `payload` | 3 | bytes | resume only |
| `useExtended` | 4 | bool | marker of extended fields |
| `extendedCiphertext` | 5 | bytes | resume with KEM |
| `paddedBytes` | 6 | bytes | padding |
| `sendServerHelloPaddedBytes` | 7 | bool | request for server-side padding |
| `simulateXxkemFs` | 8 | bool | test flag |
| `pqMode` | 9 (`PQ_MODE_FIELD_NUMBER`) | enum `C1LT` | when PQ is enabled |
| `extendedEphemeral` | 10 | bytes | the client's own ephemeral KEM key (TLV type 3) |

## 2. Full handshake (first login)

1. A new ephemeral X25519 pair is generated.
2. The public key (32 bytes) is mixed into the hash — `C1KX.A06(C27011Jk)`
   (`X/C1KX.java:183-200`), span `hash_cl_e`/`send_client_hello`: code
   `C02S.A06` and body `this.A04.A00(bArr)` — writing the ephemeral into the hash
   accumulator. The full send span is `send_client_hello` (`C1KS` code 24,
   `X/C1KS.java:53`).
3. It is placed into `ClientHello.ephemeral` (field 1).
4. With PQ enabled — `ClientHello.pqMode` (field 9, enum `C1LT`): on a full
   handshake `XXKEM`(1)/`XXKEM_FS`(2)/`XXKEM_EPH`(9) according to the variant from
   the table in the "Noise modes" chapter.
5. `static`, `payload`, `extendedCiphertext` are **not placed**: the login is sent
   later, in ClientFinish (the ClientFinish chapter). This is the main difference of
   the first login from a re-login: ServerHello responds with a certificate, and only
   after its verification does the client reveal the static key and the payload.

Frame: `HandshakeMessage { clientHello { ephemeral, [pqMode] } }`.

## 3. State machine — `C1KW`

State names (`X/C1KW.java:6-33`, two registers: `A00` — snake_case for
logs, `A01` — CamelCase):

| Code | snake_case (A00) | CamelCase (A01) | Meaning |
|-----|------------------|-----------------|-------|
| 0 | `preamble` | `Preamble` | preamble `WA\x06\x03` (+edge) |
| 1 | `client_hello` | `ClientHello` | assembly/sending of the full ClientHello |
| 2 | `client_resume` | `ClientResume` | ClientHello of the resume variant |
| 3 | `await_server_hello` | `AwaitServerHello` | waiting for ServerHello |
| 4 | `await_server_resume` | `AwaitServerHelloResume` | waiting for the resume response |
| 5 | `handle_server_hello` | `ProcessingServerHello` | parsing ServerHello |
| 6 | `handle_server_resume` | `ProcessingServerHelloResume` | parsing the resume response |
| 7 | `handle_server_fallback` | `ProcessingServerHelloFallback` | server sent static in resume |
| 8 | `client_finish` | `ClientFinish` | sending ClientFinish |
| 9 | `await_login` | `AwaitLogin` | waiting for `<success>`/`<failure>` |
| 10 | `complete` | `Complete` | login completed |
| 11 | `failed` | `Failed` | failure |

The full handshake follows the route:
`preamble(0) → client_hello(1) → await_server_hello(3) → handle_server_hello(5)
→ client_finish(8) → await_login(9) → complete(10) | failed(11)`.
The resume route: `preamble(0) → client_resume(2) → await_server_resume(4) →
handle_server_resume(6) → await_login(9)`; with static in the response —
`handle_server_fallback(7)` and a repeat pass of the full route.

## 4. Resume variant: login inside ClientHello

A re-connection when the server static is saved and the `C1KE.A01` fork
(the "Noise modes" chapter) chose resume. There is no separate ClientFinish. Into a
single `ClientHello` are placed:

1. **Mixing in the saved server static** — `C1KX.A04` (span
   `hash_svr_s`, code 17; invocation `X/C1KX.java:141-181`): the saved server X25519
   public key is mixed into the hash even before the ephemeral is sent. For
   the classic resume (IK) this step is exactly the use of the "known key" from the
   `Noise_IK_...` pattern name.
2. `ephemeral` (field 1) — a new ephemeral public key, as in the full one.
3. `static` (field 2) — the encrypted client static public key;
   **in classic IK only**. Encryption — `encrypt_cs` (span 10).
4. `payload` (field 3) — the encrypted ClientPayload (`encrypt_login_payload`,
   span 11). **In PQ mode** fields 2 and 3 are glued together: `static || ClientPayload`
   are encrypted as a single blob and placed into `payload`; the `static` field is not
   set — the server has no reason to distinguish where the key ends.
5. `pqMode` (field 9) — `IKKEM`(5)/`IKKEM_FS`(6)/`IKKEM_2`(8) per the variant.
6. `extendedCiphertext` (field 5) — if a saved server ML-KEM key exists
   (`server_static_pq_public`): the client performs an encapsulate and encodes:
   - byte `0x01` — the format version/flag;
   - 2 bytes of the encapsulate ciphertext length;
   - the ciphertext itself.
   The secret has been mixed into the key via `C1KX.A00` (MixKey).
7. `extendedEphemeral` (field 10) — for modes with the client's own ephemeral KEM key
   (`IKKEM_FS`, `XXKEM_EPH_IKKEM2`): the key is sent as a separate TLV with
   **type 0x03** (TLV structure: 1 byte type + 2 bytes length + body; types 0x01/0x02
   are taken by the server algorithm/public key, see the ServerHello chapter).

## 5. Fallback on static in the resume response

If the fork chose resume, but ServerHello nevertheless contains `static`
(for example, the server rotated its keys or did not find the session):

- the client logs `handshakeResume server hello has static key, falling back`;
- the state machine transitions to the state `handle_server_fallback` (code 7,
  `X/C1KW.java:23`);
- the current Noise context is discarded and recreated with a fallback
  pattern name from the family `Noise_XXfallback_25519_AESGCM_SHA256` /
  `Noise_XXkemfallback_...` / `Noise_XXkem-FSfallback_...` /
  `Noise_XXkemEphfallback_...` (constants `A06`, `A09`, `A0B`, `A0A`,
  `X/C1KX.java:62, 69-71`; the family selection is by the current PQ variant),
  span `init_cipher_fallback` (`C1KS` code 18, `X/C1KS.java:48`);
- then the full route `client_hello → ... → client_finish` is executed.

The new server static after a successful full handshake is saved via
`C0SW.A1d` → `AnonymousClass139.A0G` (log
`ConnectionThread/persistServerStaticKeys: server static public key changed`,
`X/C0SW.java:791-807`; `saving server static public key`,
`X/AnonymousClass139.java:466`); the PQ key — from the `static_pq_key` attribute of
the `<success>` node (the success/failure chapter).

## 6. Summary table of fields by variant

| ClientHello field | Full XX | Classic resume (IK) | Resume PQ |
|------------------|-----------|----------------------|-----------|
| `ephemeral` (1) | yes, a new pair | yes | yes |
| `static` (2) | no | yes, encrypt_cs | no (glued with payload) |
| `payload` (3) | no | yes, encrypt_login_payload | yes, = encrypt(static ‖ ClientPayload) |
| `useExtended` (4) | no | with KEM | with KEM |
| `extendedCiphertext` (5) | no | with a saved KEM key: `0x01 ‖ len16 ‖ ct` | same way |
| `paddedBytes` (6) | optional | optional | optional |
| `pqMode` (9) | XXKEM/XXKEM_FS/XXKEM_EPH | no | IKKEM/IKKEM_FS/IKKEM_2 |
| `extendedEphemeral` (10) | no | no | IKKEM_FS/XXKEM_EPH_IKKEM2: TLV type 0x03 |

The live capture of the first login is a full handshake: the FunXmpp hook does not
see ClientHello itself (it precedes the first decryption); the first visible node is
the server's response `<success t='1788392932' location='vll' pn='6615@s.whatsapp.net'
lid='4312@lid' abprops='185' creation='1788392926'/>` (02:48:52.424).
