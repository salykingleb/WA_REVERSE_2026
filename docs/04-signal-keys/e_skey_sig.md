# e_skey_sig — signed prekey signature with the identity key (curve25519sha512, 64 bytes)

`e_skey_sig` is the 64-byte signature with which the identity private
key signs the signed prekey public key. What is signed is the **33-byte
serialization WITH the 0x05 prefix** (`[0x05] ‖ skey_pub32`), although
the published `e_skey_val` itself goes without the prefix. The
algorithm is "curve25519sha512" from curve25519-java (an Ed25519-like
scheme over X25519 keys), non-deterministic (64 bytes of random nonce).
On the wire — unpadded base64url, 86 characters.

## Place in the flow

1. First installation: the signature is computed at the moment the
   signed prekey is created — `X/C0T3.java:60` — and saved in the
   protobuf record (`signature_`).
2. Registration: `X/C28804Cju.java:20-25` reads the active record; if
   `signature == null`, the entire parameter set is NOT sent — the log
   `"RegistrationKeyBundle/getKeyBundle/signedPreKey signature is null"`
   and `addKeyBundleParams/keyBundle not available` (the request goes
   out without the signal group). Otherwise the signature is added as
   the `e_skey_sig` parameter (line 33).
3. After login: the same 64 bytes go as the `<skey><signature>` node in
   the IQ `<iq xmlns='encrypt' type='set'>`
   (X/AnonymousClass132.java:472).
4. The server/peers verify the signature with the public identity key
   (`e_ident`): a match proves that the owner of the identity key
   authorized this particular signed prekey (protection against prekey
   substitution).

## Wire format (+live example)

- Raw value: exactly **64 bytes** — `R ‖ S` (32B + 32B).
- Encoding: `Base64.encodeToString(bytes, 11)` = unpadded base64url; 64
  bytes → **86 characters** (512 bits / 6 = 85.(3) → 86, no `=`).

Live example (dump of 2026-09-22, 2.26.35.75): in `/v2/code` and
`/v2/register` the parameter is second in the signal block:

```
…&e_ident=<43 симв.>&e_skey_sig=<86 символов>&entrypoint=suma&…&e_skey_val=<43 симв.>&…
```

`e_skey_sig` is byte-for-byte identical in both requests of the session
(the signature is computed once at installation and stored; it is NOT
recomputed for each request). The value is random from purchase to
purchase — 64 bytes with a ~50% high bit of S, determined by a bit of
the pub (see below).

## How it is formed in the app

### The exact signature formula

`X/C0T3.java:60`:

```java
byte[] bArrA0A = AbstractC25231B8l.A0A(c25244B8y, c25226B8g.A00());
```

- `c25244B8y` — the PRIVATE identity key (32B, from
  `identities.private_key`);
- `c25226B8g.A00()` — the 33-byte serialization of the signed prekey
  PUBLIC key WITH the prefix: `AbstractC27051Jq.A06(new byte[]{0x05},
  pub32)` (X/C25226B8g.java:17-21).

`AbstractC25231B8l.A0A` (X/AbstractC25231B8l.java:81-86) →

```java
byte[] bArrA03 = C1KK.A00("best").A03(c25244B8y.A00, bArr);
```

`C1KK.A03` (X/C1KK.java:56-62):

```java
if (bArr == null || bArr.length != 32)
    throw new IllegalArgumentException("Invalid private key length!");
return c1kl.calculateSignature(c1kl.getRandom(64), bArr /*priv*/, bArr2 /*msg*/);
```

— the provider `C1KK.A00("best")` (native libsignal-native, Java
fallback), the nonce is `getRandom(64)` = 64 fresh random bytes per
signature.

### Storage

The signature is saved in the protobuf record `signed_prekeys.record`,
field `signature_` (insert — X/C0T3.java:68; record structure — in
e_skey_id.md). When sending it is taken WITHOUT changes:
`c25233B8n.signature_.toByteArray()` (X/C0RC.java:239) — the app does
not re-sign the key on the fly.

### Reading for sending

`C0RC.A0d()` → `A03(C25232B8m)` (X/C0RC.java:235-242) assembles the
triple `C27950COg(A01=id, A00=value, A02=signature)`;
`C28804Cju.java:20-25` is the signature null-check; line 33 —
`A04("e_skey_sig", c28498Cee.A05)`.

## Native implementation

The primitive is implemented in the curve25519 provider: priority is
native libsignal-native code (inside libs.so), the fallback is pure
Java `org/whispersystems/curve25519/JavaCurve25519Provider.java:564+`
(`calculateSignature`). The scheme (per decompilation of the Java
provider and the audit):

- buffer: `bArr6[0] = 0xFE` (−2), `bArr6[1..32) = 0xFF` (−1) — the
  diversifier `0xFE ‖ 0xFF*31`; then the private key (offset 32..64),
  the message (offset 64..64+len), 64 random nonce bytes (offset
  64+len..128+len);
- `r = SHA512(diversifier ‖ priv ‖ msg ‖ random64)` (the first hash
  call over 128+len bytes);
- pub is computed from priv; `hram = SHA512(R ‖ pub ‖ msg)` (the second
  hash);
- scalar arithmetic of reduction mod l (the ed25519-order curve, a long
  chain of 21-bit limbs in the decompilation);
- output: signature = `R ‖ S`, 64 bytes; the high bit of S is adjusted
  by bit 7 of the 31st byte of the public key (in the tail code: the
  `b[31] |= (byte)(pub[63] & 128)` pattern over the output buffer),
  which is also visible in live values.

Summary: this is an Ed25519-style (XSalsa — no, SHA-512 — yes)
signature with X25519 keys from the curve25519-java library
(libsignal's axolotl heritage), and NOT classic RFC-8032 Ed25519:
different hash inputs, a random nonce instead of determinism, and a
correction of the S bit toward an "XEdDSA-like" representation.

## Value selection conditions

- The signature is computed once per record (installation or rotation);
  in repeated requests/publications the same record yields the same 64
  bytes.
- Rotation `C0RC.A0g` (X/C0RC.java:1849) signs a NEW skey_pub with the
  same identity private key — a new record, a new signature.
- Format invariants under verification: `C1KK.A01` (X/C1KK.java:36-44)
  requires the public key to be exactly 32B and the signature exactly
  64B (`bArr3.length != 64 → return false`) — the app itself validates
  foreign signatures the same way.
- What is signed is ALWAYS the 33-byte serialization with 0x05 — not
  the "bare" 32 bytes of e_skey_val.

## Cryptography and encoding

- Signature input: `msg = [0x05] ‖ skey_pub[32]` (33 bytes).
- Signing key: the private X25519 identity key (clamped 32B).
- Algorithm: curve25519sha512 (curve25519-java): a non-deterministic
  Ed25519-like one (R‖S, 64B), 64B nonce from `getRandom(64)`,
  diversifier `0xFE‖0xFF*31`, a bitwise refinement of S from the pub.
  The same (priv, msg) pair yields DIFFERENT valid signatures each
  time.
- Encoding: unpadded base64url; 64B → 86 characters. In XMPP
  `<skey><signature>` — the same bytes.

## Examples

- Live dump of 2026-09-22 (2.26.35.75): `e_skey_sig=<86 chars> in
  `/v2/code` and `/v2/register` — byte-for-byte equal (the signature
  from the DB, not a recomputation).
- Live registration of 2026-09-28 (2.26.37.73): the same geometry — 86
  base64url characters.
- Check when analyzing a dump: `len(decode(e_skey_sig)) == 64`; a
  verification attempt `verify(e_ident_pub32,
  [0x05]‖decode(e_skey_val), sig)` with the same primitive must verify
  — this is a self-check of the consistency of the
  e_ident/e_skey_val/e_skey_sig triple.
- Re-signing the same key yields a different signature (random nonce) —
  a match of signatures between different installations can never be a
  marker.
