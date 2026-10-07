# e_keytype — elliptic curve type of the key bundle (constant 0x05)

`e_keytype` is a one-byte constant `0x05` (DjbType, Curve25519/DJB) that
declares to the server the curve type of all published keys of the
bundle (`e_ident`, `e_skey_val`). After unpadded base64url encoding, the
single byte `0x05` always becomes the string **`BQ`**. This is the only
parameter of the signal group that contains no randomness at all.

## Place in the flow

- HTTP registration: added by `X/C28804Cju.java:29`
  (`c40787I4g.A04("e_keytype", c28498Cee.A02)`) inside the shared set
  of 7 signal parameters (`RegistrationKeyBundleHelper/addKeyBundleParams`).
  The call comes from `KotlinRegistrationBridge.A0U` when building each
  `/v2/code` and `/v2/register` request.
- XMPP path after login: the same byte is sent in the `<type>` node of
  the IQ `<iq xmlns='encrypt' type='set'>` when uploading prekeys:
  `new C0P3("type", new byte[]{5}, …)` (X/AnonymousClass132.java:479 —
  the full set with registration/identity/skey; :484 — a shortened one
  with only PQ keys). Live first-login capture value: the server
  receives `<type>` with the same byte 5 immediately after `<success>`
  (id `2`).
- Wire order (live dump 2.26.35.75): in `/v2/code` —
  `…,e_skey_val,authkey,e_keytype,e_regid,…`; in `/v2/register` — the
  same local order `…,e_skey_val,authkey,e_keytype,e_regid,…`.

## Wire format (+live example)

- Raw value: the array `byte[]{5}` — a static literal of the class
  `X/C28720CiW.java:6`: `public static final byte[] A02 = {5};`
- Encoding: the method shared by all signal parameters
  `C40787I4g.A04(name, bytes)` → `GCK.A0o(bytes)` →
  `android.util.Base64.encodeToString(bytes, 11)` (URL_SAFE | NO_WRAP |
  NO_PADDING).
- Calculation: the byte `0x05` = `0000 0101b`; base64 takes groups of 6
  bits: `000001` = 1 → `B`, `01 0000` (zero-padded) = 16 → `Q`. The
  result is the string `BQ`, 2 characters long, without `=`.

Live example (dump of 2026-09-22, Samsung SM-A325F, 2.26.35.75):

```
…&e_skey_val=<43 симв.>&authkey=rsgyL8y0…Azs&e_keytype=BQ&e_regid=extZ9w&…
```

`e_keytype=BQ` is present and BYTE-FOR-BYTE IDENTICAL in `/v2/code` and
`/v2/register`; in the second session (2.26.37.73, 2026-09-28) it is
also `BQ`. Any other value is impossible for a real client.

## How it is formed in the app

The value is neither computed nor stored in the DB — it is a static
constant:

1. `X/C28720CiW.java` (the full class, 9 lines):

```java
public final class C28720CiW {
    public static final byte[] A02 = {5};                       // ← сам байт
    public final InterfaceC001600s A01 = AnonymousClass057.A00(6013); // MyPreKeysManager
    public final InterfaceC001600s A00 = AnonymousClass057.A00(6015); // AuthKeyStore
}
```

2. `X/C28804Cju.java:13` takes `C28720CiW` from the lazy supplier
   (`C30513DWd.A01(12)`), and line 26 passes `C28720CiW.A02` as the
   third argument to the `C28498Cee` constructor (KeyBundleData, field
   `A02 = keyType`, confirmed by the toString class X/C28498Cee.java:
   "keyType=…").
3. Sending: `C28804Cju.java:29` → `A04("e_keytype", …)`.
4. XMPP: an independent literal `new byte[]{5}` in
   AnonymousClass132.java:479,484.

## Native implementation

There is no native side: the constant lives in a Java literal and in
the `android.util.Base64` encoding. Only the native crypto primitives
of the `C1KK.A00("best")` provider (native Curve25519 from
libsignal-native or the Java fallback) are related to it, but the type
byte itself is not passed to them — the type is implicitly fixed by the
provider choice and the length checks (32 bytes) in the calling code.

## Value selection conditions

The value is always `BQ` — there is no branching anywhere in build
2.26.35.75 (263507522). Indirect confirmations that no other type is
supported in this build:

- the external key parser `AbstractC25231B8l.A02`
  (X/AbstractC25231B8l.java:44-56) requires `bytes[0] == 5`, otherwise
  `CAI("Bad key type: " + b)`; serialization length `>= 33`;
- the signature verification `AbstractC25231B8l.A08` requires
  `c25226B8g.A00 == 5`, otherwise
  `B0W.A0r("PublicKey type is invalid")`;
- `C1KK.A01` requires the public key to be exactly 32 bytes and the
  signature exactly 64 bytes — the geometry of Curve25519;
- the public key class `C25226B8g` stores the type as the byte `A00`
  and is serialized with the prefix `[A00]‖A01`
  (X/C25226B8g.java:17-21); in all creation sites the type =
  `(byte) 5` (e.g., AbstractC25231B8l.A01:93, C0T9.A03:126).

## Cryptography and encoding

- Byte semantics: a libsignal/axolotl-style curve identifier, where `5`
  denotes DJB (Daniel J. Bernstein) X25519/Curve25519. It is the same
  prefix with which identity and signed-prekey publics are serialized
  in the DB (33 bytes = `0x05 ‖ 32B`), but in the `e_keytype` parameter
  the byte goes ALONE, separately from the keys, while in
  `e_ident`/`e_skey_val` the prefix is, on the contrary, stripped.
- Table of types observed in this build:

| Type byte | Name | Where it occurs in 2.26.35.75 | Goes on the wire |
|---|---|---|---|
| `0x05` | DjbType (Curve25519/X25519) | identity/skey serialization prefix (DB), the `C28720CiW.A02` literal, XMPP `<type>` | yes: `e_keytype=BQ`; the prefix itself is NOT sent in `e_ident`/`e_skey_val` |
| other | — | `AbstractC25231B8l.A02/A08` reject it (`Bad key type` / `PublicKey type is invalid`) | never |

- Encoding: unpadded base64url (flags 11) of a single byte; the output
  string length is always 2 characters (`BQ`).

## Examples

- `e_keytype=BQ` in the live `/v2/code` and `/v2/register` of
  2026-09-22 (2.26.35.75) — identical in both requests.
- `e_keytype=BQ` in the live registration of 2026-09-28 (2.26.37.73) —
  the value does not change between versions of the line.
- Reverse calculation: `base64url_decode("BQ") = [0x05]`;
  `base64url_decode("BQ==")` never occurs — padding is never added
  (NO_PADDING).
- XMPP equivalent after login: a `<type>` node with a body of byte 5 in
  `<iq xmlns='encrypt'>` — the same meaning, a different transport
  (AnonymousClass132.java:479).
