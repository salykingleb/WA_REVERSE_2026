# Generation and storage of the key system at first registration (summary)

Summary chapter: where in WhatsApp 2.26.35.75 (263507522) the entire key
system is created at first registration, which DI sets and classes are
responsible for it, what exactly is persisted (files/prefs/SQLite),
what is regenerated, and how the keys are related to each other.

## Place in the flow

The first installation of the app makes three independent steps (the
order between them is not guaranteed, but all of them happen before the
first `/v2/code`):

1. **Client static keypair (authkey)** — `AnonymousClass139`
   (AuthKeyStore, DI 6015): the X25519 keypair `C1Jp.A00()`, persisted
   in `shared_prefs/keystore.xml` as an encrypted blob.
2. **Signal set** — `X/C0T3.A02(SQLiteDatabase)`
   (`SignalIdentityKeyStore/createIdentityKeysAndSignedPreKeys`,
   called from the C0RC infrastructure): identity keypair +
   registration id + signed prekey (keypair+id+signature) into the
   `identities` and `signed_prekeys` tables.
3. **Registration** — `/v2/code` and `/v2/register` carry everything at
   once: the parameters are assembled by `X/C28804Cju`
   (RegistrationKeyBundleHelper), which through `C28720CiW` obtains
   both subsystems (DI 6015 — authkey, DI 6013 —
   MyPreKeysManager/reading the Signal set) and puts the 7 parameters
   (`authkey`, `e_ident`, `e_keytype`, `e_regid`, `e_skey_id`,
   `e_skey_val`, `e_skey_sig`) into the `C40787I4g` builder (uniform
   base64url-nopad encoding, flags 11). The value container is
   `C28498Cee` (KeyBundleData).

DI sets (registration in `X/C1US.java`, numbers from the
`com.whatsapp.di.*` table):

| DI id | Class | Role |
|---|---|---|
| 6013 | `X/AnonymousClass132` (MyPreKeysManager) | reading identity/regid/skey for requests; later — generation and upload of one-time and PQ prekeys (encrypt IQ) |
| 6014 | anonymous `C13G` in `X/C1US.java:15520-15550` (EncryptedKeyHelperAESPassword) | weak encryption of the static keypair (AES-OFB + PBKDF2 from the `C0UA.A0X` constant) |
| 6015 | `X/AnonymousClass139` (AuthKeyStore) | generation/storage/reading of the ClientStaticKeyPair, server static, rotation |
| 6016 | `C13A.A00()` | KeyStore helper (AndroidKeyStore AES-GCM wrapper for 6015) |
| 12 (C30513DWd) | `X/C28720CiW` | container: the 6013/6015 providers + the `{5}` literal for e_keytype |

## Wire format (+live example)

The final set on the wire (live dump of 2026-09-22, `/v2/code`, native
order): `…,e_ident,e_skey_sig,entrypoint,…,e_skey_val,authkey,e_keytype,
e_regid,…,e_skey_id,…` — all 7 values are byte-for-byte identical in
`/v2/code` and `/v2/register` of one installation.

| Parameter | Bytes | String | Value source |
|---|---|---|---|
| authkey | 32 (no 0x05) | 43 chars | `AnonymousClass139.A0D().A02.A01` — static keypair pub |
| e_ident | 32 (no 0x05) | 43 chars | `AnonymousClass132.A0Y()` → `C0RC.A1A()` — identity pub |
| e_keytype | 1 (0x05) | `BQ` | the `C28720CiW.A02` literal |
| e_regid | 4 BE | 6 chars | `AnonymousClass132.A0Z()` → `A03(C06550Sp.A06())`; live `extZ9w`=2065390071 |
| e_skey_id | 3 BE | 4 chars | `A04(record.id_)`; live `E-mO`=1304974 |
| e_skey_val | 32 (no 0x05) | 43 chars | `record` → `AbstractC25231B8l.A02` 32B slice |
| e_skey_sig | 64 | 86 chars | `record.signature_` unchanged |

## How it is formed in the app (generation and STORAGE)

### 1. Static keypair (authkey) — `AnonymousClass139` + `C1Jp.A00()`

- Generation: `C1KK.A00("best").A00.generatePrivateKey()` (32B + clamp
  `b[0]&=248; b[31]&=127; |=64`) → `generatePublicKey()` (32B, length
  invariant in `C27011Jk`). Called on first need: `A03(z)` — a
  KeyStore blob attempt (`A0B`), on failure — the pwd blob (`A05`,
  X/AnonymousClass139.java:341-354, 508-512).
- Storage: `/data/data/com.whatsapp/shared_prefs/keystore.xml`, the
  blob plaintext is 64B `priv ‖ pub` (serialization `C1Jp.A01()` /
  `AbstractC27051Jq.A06/A07`):
  - `client_static_keypair_enc` = JSON `[0, b64(ct), b64(iv)]`,
    AES-256-GCM, key `aes_auth_key` in AndroidKeyStore (spec without
    user-presence — X/C27021Jl.java:52-56);
  - `client_static_keypair_pwd_enc` = JSON `[2, b64(ct), b64(iv16),
    b64(salt4), b64(rand16)]`, AES-128-OFB, key
    PBKDF2-HMAC-SHA1(16 iter., password = the APK constant `C0UA.A0X`
    ‖ suffix) — decryptable offline (audit G, sections 2 and 7).
- The decrypted keypair is cached forever in field `A00` as raw
  `byte[]` (without zeroization). Reset: `A0E()` — removes both keys,
  log `"clearing client static key pair"`, the next access
  regenerates.
- Rotation: `X/AnonymousClass185.A00()` — IQ `w:auth:key` (at most once
  per day, when `remaining_auth_key_rotation_attempts > 0`, 30-min
  backoff); new keypair `C1Jp.A00()`, the pub goes as the `<key>` node,
  the response is saved by `X/C41722Ifn.java`.

### 2. Signal set — `X/C0T3.A02` (one creation transaction)

Sequence inside `A02(SQLiteDatabase)`:

1. `AbstractC25231B8l.A01()` — the X25519 identity keypair (clamped
   priv + pub, type 5).
2. `SecureRandom.getInstance("SHA1PRNG").nextInt(2147483646) + 1` —
   registration id (1..2147483646).
3. INSERT into `identities` (self row `recipient_id=-1,
   recipient_type=0, device_id=0`): `public_key` = 33B `[0x05]‖pub`
   (serialization `C25226B8g.A00()`), `private_key` = raw 32B WITHOUT
   column encryption, `registration_id`,
   `next_prekey_id`/`next_kyber_prekey_id` = `A00()+1` (start:
   `nextInt(16777214)` without +1), `timestamp`. Log
   `"SignalIdentityKeyStore/inserted identity key pair"`.
4. `AbstractC42871v4.A00()` (SDK>=26 →
   `SecureRandom.getInstanceStrong()`) → `nextInt(16777214) + 1` —
   signed prekey id.
5. The second keypair `AbstractC25231B8l.A01()` — signed prekey;
   signature `AbstractC25231B8l.A0A(identityPriv, [0x05]‖skeyPub33)` —
   curve25519sha512 (see e_skey_sig.md).
6. INSERT into `signed_prekeys`: `prekey_id`, `timestamp`, `record` =
   protobuf `C25233B8n{id_, publicKey_ 33B, privateKey_ 32B,
   signature_ 64B, timestamp_}`. Log
   `"SignalIdentityKeyStore/inserted signed prekey"` +
   `"SignalCoordinator/createIdentityKeysAndSignedPreKeys generated
   random starting ID: N"`.

### 3. What appears LATER (after the first login, not in HTTP registration)

- One-time prekeys (X25519) and PQ ML-KEM kyber prekeys: generated and
  uploaded by `AnonymousClass132` (uploadNextBatch) as the `<list>`,
  `<pq_list>`, `<pq_last_resort_key>` nodes in the IQ
  `<iq xmlns='encrypt'>`; ids from the
  `next_prekey_id`/`next_kyber_prekey_id` counters.
- The server static Noise key: arrives in the first full handshake,
  saved by `AnonymousClass139.A0G` to the prefs `server_static_public`
  (base64, flags 3); with it, subsequent logins go as resume-IK. The
  server's PQ key is `server_static_pq_public` (`A0H`).
- Signed prekey rotation: `C0RC.A0g/B8V` (new keypair+signature, id =
  last + rand with wrap `((x-1)%16777214)+1`), the new record becomes
  active.

## Native implementation

- All Curve25519 operations (keypair generation, DH, signatures) — via
  `C1KK.A00("best")` = `OpportunisticCurve25519Provider`: priority is
  native libsignal-native (in libs.so), fallback
  `JavaCurve25519Provider` (org/whispersystems/curve25519/*, pure
  Java).
- The private half of the static keypair is passed to the wamsys native
  layer: `JniBridge` case 3 (com/whatsapp/wamsys/JniBridge.java
  ~476-484) reads `A0D().A01.A01`, checks `length == 32`, error
  `"AuthKeyStoreImpl/the key length is not expected/privateLength=N"`.
- The private identity/skey keys do not go to native — they are
  consumed by the Java Signal stack (C0RC and below). The 0x05 prefix,
  serialization, BE packing of ids are pure Java (`C25226B8g`,
  `AbstractC27051Jq`).

## Value selection conditions / what is regenerated

| Object | When created | Regeneration |
|---|---|---|
| Static keypair (authkey) | first installation (lazily, on first access) | only reset `A0E()` or server rotation `w:auth:key` |
| Identity keypair + regid | first installation, `C0T3.A02` | NEVER within an installation; only a clean reinstall (deleting data) |
| Signed prekey (id+keypair+signature) | first installation, `C0T3.A02` | rotation `C0RC.A0g` (new record, the last by `_id` is active) |
| One-time/PQ prekeys | after login, in batches | constantly, as they are consumed (uploadNextBatch) |
| Server static / server PQ | first full Noise handshake | on fallback events (corrupted frame → `A0C` forgets the static → full XX again) |

Static keypair storage blob choice: the vendor blacklist (AB flag 388 —
KeyStore is disabled entirely), verifying stage (AB 831/378 — both
copies in parallel; on the live testbed of 2.26.38 the pwd copy was NOT
removed even after 189 successful verifications), the failure counter
`can_user_android_key_store` and the `"failed too much must recover"`
path.

## Cryptography and encoding

- The curve of all keypairs is X25519 (DjbType 0x05); private keys
  clamped; pub exactly 32B.
- Signatures — curve25519sha512 (Ed25519-like over X25519,
  non-deterministic, 64B): e_skey_sig (identity → skey_pub33),
  vname/VerifiedName (identity), PQ keys.
- Wire encoding is uniform: `Base64.encodeToString(bytes, 11)` =
  unpadded base64url (32B→43, 4B→6, 3B→4, 1B→2, 64B→86 characters).
- Encoding in storages: DB — 33B with 0x05 for publics, raw bytes for
  privates; prefs — base64 with standard flags 3
  (`NO_WRAP|NO_PADDING`) for server static; static keypair blobs —
  AES-GCM/AES-OFB of 64B `priv‖pub`.
- The derived integrity envelope key: `AES =
  SHA256(std_base64(authkey))` (live: `rsgyL8y0…Azs` →
  `R41B51Od…JK4=`; `Me_A3cdy…b2w` → `pjhfD73u…5qo=`) — decrypts
  gpia/_gi/_gg/integrity_payload/jws.

## Examples

Text diagram of the key system relationships:

```
FIRST INSTALLATION
├─ AnonymousClass139 (DI 6015, prefs "keystore.xml")
│   └─ C1Jp.A00(): static X25519 (priv32‖pub32, encrypted)
│       ├─ pub → authkey parameter (/v2/code, /v2/register)
│       ├─ pub → SHA256(std_b64) → AES key of integrity blobs
│       ├─ pub → Noise ClientFinish.static (first login) /
│       │        ClientHello.static (resume IK); the server identifies
│       │        the account by (username + static)
│       └─ priv → DH in the handshake; → native (JniBridge case 3)
│           rotation: AnonymousClass185 → IQ w:auth:key → C41722Ifn
│
└─ C0T3.A02 (SQLite, Signal set)
    ├─ identities (self -1/0/0)
    │   ├─ identity X25519 (pub 33B with 0x05 / priv 32B plaintext)
    │   │   ├─ pub[1:] → e_ident parameter; XMPP <identity>
    │   │   └─ priv → signs skey_pub (→ e_skey_sig) and vname
    │   ├─ registration_id = SHA1PRNG.nextInt(2147483646)+1
    │   │   └─ 4B BE → e_regid parameter; XMPP <registration>
    │   └─ next_prekey_id / next_kyber_prekey_id (OTP/PQ counters)
    │
    └─ signed_prekeys (protobuf C25233B8n, active = last _id)
        ├─ id = StrongRandom.nextInt(16777214)+1 → 3B BE → e_skey_id
        ├─ pub X25519 → 33B in the record, 32B on the wire → e_skey_val
        └─ signature = curve25519sha512(identityPriv, [0x05]‖skeyPub)
            → 64B → e_skey_sig
            rotation: C0RC.A0g (id = last+rand, wrap 16777215→1)

AFTER LOGIN (not in HTTP registration)
├─ AnonymousClass132.uploadNextBatch → IQ encrypt: <type>{5},
│   <registration>, <identity>, <skey id/value/signature>,
│   <list> (one-time X25519), <pq_list>/<pq_last_resort_key> (ML-KEM)
└─ AnonymousClass139.A0G/A0H → server_static_public /
    server_static_pq_public in prefs (for the resume handshake)
```

Live cross-check (dump of 2026-09-22): authkey 43 chars, e_keytype
`BQ`, e_regid `extZ9w`=2065390071 (first byte 0x7B — high bit 0),
e_skey_id `E-mO`=1304974 — all parameters are consistent with the rules
described above.
