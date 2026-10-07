# e_skey_val — signed prekey public key (X25519, 32 bytes without prefix)

`e_skey_val` is the public half of the signed prekey keypair (one-time
in meaning, but the long-lived "last resort" prekey of the Signal
protocol), generated at first installation and signed with the identity
key (the signature is the neighboring parameter `e_skey_sig`). Exactly
32 bytes of the public key go on the wire WITHOUT the 0x05 prefix,
unpadded base64url (43 characters).

## Place in the flow

1. First installation: `X/C0T3.java:58` generates a new X25519 keypair
   with the same method as the identity (`AbstractC25231B8l.A01()`),
   signs the public key with the identity private key and writes a
   protobuf record to `signed_prekeys`.
2. Registration: `X/C28804Cju.java:32` reads the active record
   (`AnonymousClass132.A0L()` → `C0RC.A0d()` → `A03(...)`, the `A00`
   value of the `C27950COg` triple) and adds it as the `e_skey_val`
   parameter.
3. After login: the same 32 bytes go as the `<skey><value>` node in the
   IQ `<iq xmlns='encrypt' type='set'>` (X/AnonymousClass132.java:472)
   — live first-login capture, id `2`, immediately after `<success>`.
4. On rotation (long-lived account): `C0RC.A0g` creates a new
   keypair+signature, the new record becomes active — from that moment
   the new key appears in `<skey>` and any repeated publications. In
   the HTTP registration of a fresh installation the original record
   from C0T3 applies.

## Wire format (+live example)

- Value: exactly **32 bytes** of the X25519 public key **WITHOUT the
  0x05 prefix**, `Base64.encodeToString(bytes, 11)` → **43 characters**
  of base64url without `=`.
- In the DB protobuf record (`C25233B8n.publicKey_`) the key is stored
  as 33 bytes WITH the `0x05` prefix (`c25226B8g.A00()`); when reading
  for sending the prefix is strictly cut off (see below).

Live example (dump of 2026-09-22, 2.26.35.75): in `/v2/code` and
`/v2/register` the parameter stands in one block with the other signal
ones:

```
…&e_skey_val=<43 символа base64url>&authkey=rsgyL8y0…Azs&e_keytype=BQ&e_regid=extZ9w&…&e_skey_id=E-mO&…
```

`e_skey_val` is byte-for-byte identical in both requests of the session
(2026-09-22) and in the session of 2026-09-28 (2.26.37.73) — a fresh
installation does not change it between requests. The value is random
32 bytes, with no distinguishable prefixes.

## How it is formed in the app

### Keypair generation (first installation)

`X/C0T3.java:58` (inside `A02`):

```java
C25243B8x c25243B8xA02 = AbstractC25231B8l.A01();   // новая X25519-пара, тип 5
```

`AbstractC25231B8l.A01()` (X/AbstractC25231B8l.java:88-96) — the same
provider `C1KK.A00("best")`: `generatePrivateKey()` (32B + clamp
`b[0]&=248; b[31]&=127|=64`, JavaCurve25519Provider.java:410-421) →
`generatePublicKey(priv)` → `C25226B8g(pub32, (byte)5)`. Then:

```java
byte[] sig = AbstractC25231B8l.A0A(identityPriv, spreKeyPub33); // подпись (см. e_skey_sig)
builder.A00(iNextInt2);                       // id
builder.A03(ByteString.copyFrom(pub33));      // publicKey_ = [0x05]||pub32
builder.A02(ByteString.copyFrom(priv32));     // privateKey_
builder.A04(ByteString.copyFrom(sig));        // signature_
```

and `insertOrThrow("signed_prekeys", …)` (X/C0T3.java:61-75) with the
log `"SignalIdentityKeyStore/inserted signed prekey"`.

### Storage

Table `signed_prekeys`, protobuf `C25233B8n` in the `record` column
(structure in e_skey_id.md): public — 33B with `0x05`, private — raw
32B, signature — 64B. Active record — the last by `_id` (`C0RC.A0a`).

### Reading for sending (where 0x05 is cut off)

`C0RC.A0d()` (X/C0RC.java:2964-2968) → `A0a()` (active record) →
`A03(C25232B8m)` (X/C0RC.java:235-242):

```java
byte[] bArr = c25232B8m.A00().A01.A01;   // 32 байта без префикса
```

where `C25232B8m.A00()` parses `record` via
`AbstractC25231B8l.A02(bytes)` (X/AbstractC25231B8l.java:44-56):

```java
if (bArr.length < 33) throw new CAI("Invalid byte array");
if (bArr[0] != 5)     throw new CAI("Bad key type: " + b);
byte[] bArr2 = new byte[32];
System.arraycopy(bArr, 1, bArr2, 0, 32);   // ← префикс отброшен
return new C25226B8g(bArr2, (byte) 5);
```

## Native implementation

Keypair generation — via `C1KK.A00("best")` =
`OpportunisticCurve25519Provider` (native Curve25519 from
libsignal-native in libs.so; fallback — `JavaCurve25519Provider`).
Storage and record parsing are pure Java (protobuf classes, SQLite).
The key is not passed to the wamsys native layer: it is needed by the
Java Signal stack when establishing sessions with peers (when a peer
takes this prekey from the bundle on the server).

## Value selection conditions

- One active keypair at the moment of registration: the `/v2/code` and
  `/v2/register` requests feature the record inserted at installation.
- Rotation `C0RC.A0g` (X/C0RC.java:1847-1849): a new
  `AbstractC25231B8l.A01()` keypair + a new signature with the identity
  key + id by the increment/wrap rule; the new record becomes active,
  the old ones remain in the DB.
- The format is invariant: 32 bytes (a 33-byte value on the wire is a
  parsing error on the server side / a marker of a foreign
  implementation).
- One-time prekeys (`list` in the encrypt IQ) and PQ/ML-KEM kyber
  prekeys (`pq_list`, `pq_last_resort_key`) exist separately — they are
  NOT part of HTTP registration, they have their own keys and ids.

## Cryptography and encoding

- Curve: X25519 (Curve25519, DJB), private key clamped, public 32B
  little-endian.
- Purpose: the "last resort" prekey — a peer builds the initial session
  from the combination of `e_ident` + `e_skey_val` + `e_skey_sig` (+
  one-time ones if available), hence the requirements: the key must be
  a valid X25519 point (the provider rejects small-subgroup on read),
  the signature must verify under the identity key.
- Encoding: unpadded base64url (flags 11); 32B → 43 characters. In the
  DB and protobuf — 33B with `0x05`; in XMPP `<skey><value>` — the
  same 32B without the prefix.

## Examples

- Live dump of 2026-09-22 (2.26.35.75): `e_skey_val=<43 chars> in
  `/v2/code` and `/v2/register`, the values match; alongside
  `e_skey_id=E-mO`, `e_keytype=BQ`, `e_regid=extZ9w`.
- Live registration of 2026-09-28 (2.26.37.73): the same 43-character
  format, part of the immutable part of the requests together with
  authkey/e_ident.
- Format check when analyzing dumps:
  `len(base64url_decode(e_skey_val)) == 32` always; 33 bytes with the
  first being 0x05 mean an encoding error (the prefix was not
  stripped).
- The bundle in the encrypt IQ: `<skey><id>`(=e_skey_id)`<value>`
  (=e_skey_val)`<signature>`(=e_skey_sig) — the same bytes as in HTTP,
  a different transport (AnonymousClass132.java:472).
