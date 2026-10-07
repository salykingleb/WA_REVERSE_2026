# e_skey_id — signed prekey identifier (3 bytes big-endian, 1..16777214)

`e_skey_id` is the numeric identifier of the signed prekey created at
first installation together with the identity keypair. It goes on the
wire as 3 bytes **big-endian** in unpadded base64url (4 characters). The
range is the full 24-bit one without zero: 1..16777214 (0xFFFFFF is
reserved as "invalid").

## Place in the flow

1. First installation: `X/C0T3.java:53-55` generates the id
   `SecureRandom(strong).nextInt(16777214) + 1` and, together with the
   prekey keypair itself and the signature, writes a protobuf record to
   the SQLite table `signed_prekeys`.
2. Registration: `X/C28804Cju.java:19,31` reads the ACTIVE (latest)
   record via `AnonymousClass132.A0L()` → `C0RC.A0d()` →
   `A03(C25232B8m)` and adds it as the `e_skey_id` parameter.
3. After login: the same id goes as the `<skey><id>…` node in the IQ
   `<iq xmlns='encrypt' type='set'>` (X/AnonymousClass132.java:472).
4. On signed prekey rotation (already after login, `C0RC.A0g`) a NEW
   record is created with id = old + a random increment (with wrap), and
   the new id then appears in requests — but in the HTTP registration
   of a fresh installation it is exactly the random path from C0T3 that
   applies.

## Wire format (+live example)

- Raw value: int in the range **1 .. 16777214** (24 bits, 0x00FFFFFF
  excluded).
- Serialization: `AbstractC27051Jq.A04(i)`
  (X/AbstractC27051Jq.java:18-20):

```java
return new byte[]{(byte)(i >> 16), (byte)(i >> 8), (byte) i};
```

— 3 bytes **big-endian**.

- Encoding: unpadded base64url (flags 11); 3 bytes → exactly **4
  characters**, without `=` (24 bits divide by 6 without a remainder).

Live example (dump of 2026-09-22, 2.26.35.75):

```
e_skey_id=E-mO
```

Decoding check: `E=4, -=62, m=38, O=14` → bits
`000100 111110 100110 001110` → bytes `0x13 0xE9 0x8E` →
**0x13E98E = 1304974**. The value is in the range 1..16777214, the high
byte 0x13 is arbitrary (the range does NOT constrain the high bit,
unlike e_regid). The value is identical in `/v2/code` and
`/v2/register` of one session.

## How it is formed in the app

### Generation (first installation)

`X/C0T3.java:53-55`:

```java
SecureRandom secureRandomA00 = AbstractC42871v4.A00();
C000800h.A06(secureRandomA00);
int iNextInt2 = secureRandomA00.nextInt(16777214) + 1;
```

- `AbstractC42871v4.A00()` (X/AbstractC42871v4.java:10-19): on SDK >= 26
  — `SecureRandom.getInstanceStrong()` (the strongest OS provider
  available), on `NoSuchAlgorithmException` or on older SDKs —
  `new SecureRandom()`. This is a DIFFERENT RNG than for `e_regid`
  (where an explicit SHA1PRNG is used).
- `nextInt(16777214)` → 0..16777213, `+1` → 1..16777214: a zero id is
  impossible, 0xFFFFFF (16777215) is never produced.
- Post-insert log: `"SignalCoordinator/createIdentityKeysAndSignedPreKeys
  generated random starting ID: N"` (X/C0T3.java:77-80).

### Storage

Table `signed_prekeys`, insert at `X/C0T3.java:71-75`:

| Column | Contents |
|---|---|
| `prekey_id` | the id itself (int) |
| `timestamp` | creation time (sec) |
| `record` | protobuf `C25233B8n`: `id_`, `publicKey_` (33B with 0x05), `privateKey_` (32B), `signature_` (64B), `timestamp_` |

The record with the highest `_id` (the last inserted) is considered
active — `C0RC.A0a()` loads it for sending; `C0RC.A03(C25232B8m)`
(X/C0RC.java:235-242) parses the protobuf and assembles the
id/value/signature triple (`C27950COg`: `A01`=id, `A00`=value,
`A02`=signature).

### Rotation (after login)

`X/C0RC.java:1820-1841`: new id = `last prekey_id + a random increment
i`; if the sum is `>= 16777215` — wrap by the formula
`((x - 1) % 16777214) + 1` (line 1833). Old records are not deleted
immediately — the new one becomes active. For a FRESH installation
(HTTP registration) this path does not apply — there only the random
starting id from C0T3 applies.

## Native implementation

There is no native side: the id is an int in SQLite and protobuf,
serialization by `AbstractC27051Jq.A04` and encoding by
`android.util.Base64` are pure Java.
`SecureRandom.getInstanceStrong()` on modern Android relies on the OS's
native entropy source (getrandom), but that is a system mechanism, not
libsignal-native.

## Value selection conditions

- First installation: uniformly random [1, 16777214].
- Rotations: monotonic growth with wrap (`last + rand`, wrap 16777215 →
  1), i.e. on a long-lived account values tend to be "following the
  previous one"; this is not visible at registration.
- Never: 0 and 16777215 (0xFFFFFF). Both edges are statistical markers
  of a foreign implementation when analyzing dumps.
- Distinguish from `next_prekey_id` (one-time prekeys): that one lives
  in `identities.next_prekey_id`, starts from `nextInt(16777214)`
  without +1 and does not get into HTTP registration.

## Cryptography and encoding

- The id itself is an identifier, not a secret; its randomness is
  needed only for uniqueness/unlinkability.
- Byte order: big-endian (the most significant of the three first) —
  `A04`, the mirror of the parser `A00(bytes)`
  (X/AbstractC27051Jq.java:10-12).
- Encoding: unpadded base64url; 3B → 4 characters; the range of the
  first character is unconstrained (A.._), since the high bit is
  allowed.
- Together with `e_skey_val`/`e_skey_sig` it forms the XMPP `<skey>`
  node: the same bytes (`id`/`value`/`signature`), with no base64
  differences.

## Examples

- Live: `e_skey_id=E-mO` → 0x13E98E → 1304974 (dump of 2026-09-22,
  Samsung SM-A325F, 2.26.35.75; identical in `/v2/code` and
  `/v2/register`).
- Encoding the boundaries: id 1 → `[0x00 0x00 0x01]` → `AAAB`; id
  16777214 → `[0xFF 0xFF 0xFE]` → bits `111111 111111 111111 111110`
  → indices 63,63,63,62 → the string `___-` (the range maximum; the
  value 0xFFFFFF would encode as `____`, but the app never produces
  it).
- XMPP equivalent: `<skey><id>`(3B BE)`<value>`(32B)`<signature>`(64B)
  — exactly the same triple of protobuf record fields
  (AnonymousClass132.java:472).
- Rotation: a record with `prekey_id = 16777213 + 5` → wrap
  `((16777217-1) % 16777214)+1 = 3` — this is how the server receives
  a "lower" id after overflow (C0RC.java:1833).
