# e_ident — Signal public identity key (X25519, 32 bytes without prefix)

`e_ident` is the public half of the Signal protocol (libsignal/axolotl)
identity keypair, generated once at first installation. It is the
account's "identity" in the E2E protocol: the signed prekey is signed
with it (`e_skey_sig`), sessions with peers are established with it, and
it is also sent to the server at registration as the `e_ident` parameter
and in the `encrypt` IQ after login. The private half NEVER leaves the
device.

## Place in the flow

1. First installation: `X/C0T3.java` (`SignalIdentityKeyStore/
   createIdentityKeysAndSignedPreKeys`, method `A02`) generates the
   identity keypair and writes it to the SQLite table `identities` (self
   row `recipient_id=-1, recipient_type=0, device_id=0`).
2. Registration: `X/C28804Cju.java:17` reads the key via
   `AnonymousClass132.A0Y()` (MyPreKeysManager, DI 6013) → `C0RC.A1A()`
   and adds it as the `e_ident` parameter (line 28) to `/v2/code` and
   `/v2/register`.
3. After login: the same 32-byte public key goes as the `<identity>`
   node in the IQ `<iq xmlns='encrypt' type='set'>`
   (X/AnonymousClass132.java:463) — in the live first-login capture this
   is id `2`, immediately after `<success>`.
4. The rest of the account's life: the identity key participates in
   safety-number checks/QR scanning with peers, in signing vname
   (Verified Name, business profile) with the same curve25519sha512
   primitive using the private half (C1KK.A03), and in session-build
   with peers (C0RC.processPreKeyBundle).

## Wire format (+live example)

- Value: **exactly 32 bytes** of the X25519 public key **WITHOUT the
  0x05 type prefix**, encoded with `Base64.encodeToString(bytes, 11)` =
  unpadded base64url → a string always **43 characters** long (as with
  `authkey`).
- In the DB and inside protobuf records the key is stored as 33 bytes
  WITH the `0x05` prefix (`C25226B8g.A00()` = `[0x05] ‖ pub32`), but
  the prefix does not reach the wire: the read for sending returns
  exactly the `A01` field (the bare 32 bytes).

Live example (dump of 2026-09-22, 2.26.35.75): in `/v2/code` and
`/v2/register` the parameter stands next to the other signal ones:

```
…&e_ident=<43 символа base64url>&e_skey_sig=<86 символов>&…&e_skey_val=<43 символа>&authkey=rsgyL8y0…Azs&…
```

`e_ident` is byte-for-byte identical in both requests of one session
(like all signal parameters, except the recreated
`_gs/_gg/gpia/_gi/client_metrics`). It does not start with a fixed
prefix — these are purely random 32 bytes.

## How it is formed in the app

### Keypair generation (first installation)

`X/C0T3.java:28-29` inside `A02(SQLiteDatabase)`:

```java
C25243B8x c25243B8xA01 = AbstractC25231B8l.A01();
C25230B8k c25230B8k = new C25230B8k(c25243B8xA01.A00, new C25224B8e(c25243B8xA01.A01));
```

`AbstractC25231B8l.A01()` (X/AbstractC25231B8l.java:88-96):

```java
C1KL c1kl = C1KK.A00("best").A00;
byte[] priv = c1kl.generatePrivateKey();          // 32B случайных + clamp
byte[] pub  = c1kl.generatePublicKey(priv);       // X25519 ScalarBaseMult
return new C25243B8x(new C25244B8y(priv), new C25226B8g(pub, (byte) 5));
```

- private key clamp: `b[0] &= 248; b[31] &= 127; b[31] |= 64`
  (JavaCurve25519Provider.java:410-421);
- entropy — 256 random bits (native libsignal-native provider, Java
  fallback);
- the key type is fixed as `(byte) 5` (DjbType).

### Storage

SQLite table `identities`, self row (`recipient_id=-1,
recipient_type=0, device_id=0`), insert at `X/C0T3.java:36-47`:

| Column | Contents | Format |
|---|---|---|
| `public_key` | public identity key | 33 bytes `[0x05] ‖ pub32` (serialization `C25230B8k.A01.A00.A00()`) |
| `private_key` | private identity key | raw 32 bytes, WITHOUT column-level encryption |
| `registration_id` | see e_regid | int |
| `next_prekey_id`, `next_kyber_prekey_id`, `timestamp` | service fields | int/int/long |

Insert log: `"SignalIdentityKeyStore/inserted identity key pair"`. The
keys are NOT regenerated on repeated registrations of the number — only
on a clean installation (deleting the app data).

### Reading for sending

- HTTP: `AnonymousClass132.A0Y()` (X/AnonymousClass132.java:833-848) →
  `C0RC.A1A()` (X/C0RC.java:2776-2782): `this.A01.A03().A01.A00.A01` —
  the bare 32 bytes of `C25226B8g.A01`; log
  `"SignalCoordinator/fetched identity key for sending"`. Storage
  decoding: `C0T9.A03()` (X/C0T9.java:124-127) parses the 33-byte
  record and restores `C25226B8g(pub32, (byte)5)`.
- XMPP: the same `A0Y()` goes into `<identity>`
  (AnonymousClass132.java:463).

## Native implementation

- Generation/signature/agreement — via `C1KK.A00("best")` →
  `OpportunisticCurve25519Provider` (native libsignal-native in libs.so,
  Java fallback `JavaCurve25519Provider`). The identity key itself is
  not passed to native: signatures and sessions are computed at the
  Java level of the Signal stack (`C0RC`, `AbstractC25231B8l`,
  `C1KK`).
- The private half is stored in SQLite without file-level encryption
  (audit G_client_static_keypair_extraction.md, section 3): if
  `/data/data` is compromised, the entire installation's Signal
  identity leaks at once.

## Value selection conditions

- One value per installation: between `/v2/code` and `/v2/register`,
  between restarts and repeated logins — the same 32 bytes.
- The format is strictly checked: `C1KK.A01` requires the public key to
  be exactly 32 bytes; `AbstractC25231B8l.A02` when reading from the DB
  requires length `>= 33` and `bytes[0] == 5` (`"Bad key type"`), after
  which it cuts off exactly 32 bytes.
- The `0x05` prefix never appears on the wire (field `A01`, not
  `A00()`).

## Cryptography and encoding

- Curve: Curve25519 (X25519, DJB) — the same as for `authkey` and
  `e_skey_val`, but with a different role: the identity key is
  long-lived, signs other keys and is recognized by peers.
- Signing with the identity private half: curve25519sha512 (an
  Ed25519-like scheme over X25519 keys from curve25519-java) — used for
  `e_skey_sig` (signature over `[0x05]‖skey_pub33`) and for
  vname/VerifiedName (the same `C1KK.A03` primitive; in consumer
  registration the `vname` parameter is absent — the signature appears
  in business flows and in the iq encrypt after login).
- Wire encoding: unpadded base64url (flags 11), 32B → 43 characters. In
  the DB: 33B with the prefix. In XMPP `<identity>`: the same 32 bytes
  without the prefix.

## Examples

- Live dump of 2026-09-22 (2.26.35.75): `e_ident=<43 chars> in
  `/v2/code` and `/v2/register`, the values match byte-for-byte.
- Live registration of 2026-09-28 (2.26.37.73): the same 43-character
  format; together with `authkey` and `e_skey_*` it is part of the
  immutable part of the requests.
- The bundle in the IQ after the first login: `<identity>`(32B pub) +
  `<registration>`(4B BE) + `<skey><id/><value/><signature/>` +
  `<type>`(0x05) — all from the same storages
  (AnonymousClass132.java:451-479).
- Format check: decode the `e_ident` value → it must come out as
  exactly 32 bytes; if 33 and the first is `0x05` — that is DB
  serialization, which must not be like that on the wire.
