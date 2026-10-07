# e_regid — Signal protocol registration id (4 bytes big-endian, 1..2147483646)

`e_regid` is a 32-bit Signal registration identifier generated once at
first installation from a cryptographic RNG and saved in the DB next to
the identity key. It goes on the wire as 4 bytes **big-endian** in
unpadded base64url (6 characters). The value range is strictly limited
to positive int32: the high bit of the first byte is ALWAYS 0.

## Place in the flow

1. First installation: `X/C0T3.java:35` generates the value
   `SecureRandom.getInstance("SHA1PRNG").nextInt(2147483646) + 1` and
   writes it to `identities.registration_id` (self row
   `recipient_id=-1,0,0`) together with the identity keypair.
2. Registration: `X/C28804Cju.java:18` reads the value via
   `AnonymousClass132.A0Z()` → `C06550Sp.A06()` and adds it as the
   `e_regid` parameter (line 30) to `/v2/code` and `/v2/register`.
3. After login: the same value goes as the `<registration>` node in the
   IQ `<iq xmlns='encrypt' type='set'>` (X/AnonymousClass132.java:462) —
   in the live first-login capture this is id `2`.
4. Further role: the registration id is part of the account's prekey
   bundle and lets peers/the server distinguish a reinstall (new regid)
   from the same device.

## Wire format (+live example)

- Raw value: int32 in the range **1 .. 2147483646** (2^31 − 2).
- Serialization: `AbstractC27051Jq.A03(i)`
  (X/AbstractC27051Jq.java:14-16):

```java
return new byte[]{(byte)(i >> 24), (byte)(i >> 16), (byte)(i >> 8), (byte) i};
```

— 4 bytes **big-endian**.

- Encoding: `Base64.encodeToString(bytes, 11)` = unpadded base64url; 4
  bytes → **6 characters** (32 bits / 6 = 5.(3) → 6, the last 4 bits
  are discarded as padding zeros — therefore the value is unambiguously
  restored from the 6 characters: `decode → first 4 bytes`).

Live example (dump of 2026-09-22, 2.26.35.75):

```
e_regid=extZ9w
```

Decoding check: `e=30, x=49, t=45, Z=25, 9=61, w=48` → bits
`011110 110001 101101 011001 111101 110000` → bytes `0x7B 0x1B 0x59
0xF7` (the trailing `0000` is padding) → **0x7B1B59F7 = 2065390071**.
The value lies within the app's range (2065390071 ≤ 2147483646), the
first byte `0x7B < 0x80` — the high bit is zero, as required. The value
is identical in `/v2/code` and `/v2/register` of one session.

## How it is formed in the app

### Generation (once per installation)

`X/C0T3.java:35-40`:

```java
int iNextInt = SecureRandom.getInstance("SHA1PRNG").nextInt(2147483646) + 1;
...
contentValues.put("recipient_id", (Integer) (-1));
contentValues.put("recipient_type", (Integer) 0);
contentValues.put("device_id", (Integer) 0);
contentValues.put("registration_id", Integer.valueOf(iNextInt));
...
sQLiteDatabase.insertOrThrow("identities", null, contentValues);
```

- RNG: `SecureRandom.getInstance("SHA1PRNG")` — an explicit
  provider-specific implementation (on Android it is served by OpenSSL
  entropy).
- `nextInt(2147483646)` yields 0..2147483645, `+1` shifts it to
  1..2147483646: the value is never equal to 0 and never has the sign
  bit set (first byte ≤ 0x7F). This is a strict statistical
  characteristic of real clients: the real app never produces values
  with a first byte of 0x80..0xFF.

### Storage

- The `registration_id` column of the `identities` table, self row
  `recipient_id=-1, recipient_type=0, device_id=0`; the value does not
  change until the app data is deleted.
- In-memory cache: `C06550Sp.A00` (int), reset when the process
  restarts.

### Reading for sending

`C06550Sp.A06()` (X/C06550Sp.java:513-527):

```java
Cursor c = ...A0A("SELECT registration_id FROM identities WHERE recipient_id =?
                    AND recipient_type = ? AND device_id = ?",
                   "SignalIdentityKeyStore/getRegistrationId",
                   new String[]{String.valueOf(-1), "0", "0"});
if (!c.moveToNext()) throw new SQLiteException("Missing entry for self in identities table");
```

then `AnonymousClass132.A0Z()` (X/AnonymousClass132.java:850-865) calls
`AbstractC27051Jq.A03(regId)` — those very 4 BE bytes.

## Native implementation

There is no native side: the value is an int in SQLite, serialization
and encoding are pure Java (`AbstractC27051Jq`,
`android.util.Base64`, flags 11). The SHA1PRNG RNG is a JVM/Android
provider and has nothing to do with libsignal-native.

## Value selection conditions

- Uniformly random from [1, 2147483646]; no constants/counters.
- Not recreated between `/v2/code` → `/v2/register` → logins; only a
  clean installation yields a new value.
- A related but DIFFERENT value: `next_prekey_id` in the same table is
  computed as `SecureRandom.getInstance("SHA1PRNG").nextInt(16777214)`
  WITHOUT `+1` (X/C0T3.java:97-103, range 0..16777213) — it is the
  starting counter of one-time prekeys; it is not sent in HTTP
  registration.
- The XMPP path uses the same value from the same DB row (the
  `<registration>` node); there is no separate copy.

## Cryptography and encoding

- Source of randomness: SHA1PRNG SecureRandom — a cryptographically
  strong PRNG seeded with system entropy; the registration id itself is
  not a secret (it is an identifier), but it has no correlation with
  time/device.
- Byte order: big-endian (most significant byte first) — `A03`, the
  mirror of the parser `A01(bytes, offset)`.
- Encoding: unpadded base64url; 4 bytes → 6 characters; alphabet range
  `A-Z a-z 0-9 - _`. A consequence of the 1..0x7FFFFFFE range: the
  first 6 bits of the value are ≤ 31, so the FIRST character of a real
  client's e_regid string always lies in `A..Z, a..f` (alphabet indices
  0..31) — the characters `g..z, 0..9, -, _` can never stand first. The
  live `extZ9w` starts with `e` (index 30) — within bounds.
- Difference from `e_skey_id`: there it is 3 BE bytes and a different
  RNG/range (see e_skey_id.md).

## Examples

- Live: `e_regid=extZ9w` → 0x7B1B59F7 → 2065390071 (dump of 2026-09-22,
  `/v2/code` and `/v2/register`, Samsung SM-A325F, 2.26.35.75).
- Another live dump from the keyhelper audit notes: 1634972141 — also
  < 2147483646 and > 0, confirming the range at a second point.
- Encoding the number 1: `A03(1)` = `[0x00 0x00 0x00 0x01]` →
  `AAAAAQ`; the number 2147483646: `[0x7F 0xFF 0xFF 0xFE]` → `f____-`
  (the range boundary).
- Anti-example (a real client does NOT do this): a fully random uint32
  could give a first byte of 0x80..0xFF (a 6-character string starting
  with w.._ or with `-`/`_` in the high positions) or the value 0 —
  such `e_regid` values are identified as foreign on the wire.
