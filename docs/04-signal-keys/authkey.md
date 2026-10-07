# authkey — public key of the client's static X25519 keypair (ClientStaticKeyPair)

`authkey` is the public half (exactly 32 bytes, WITHOUT the 0x05 type
prefix) of the device's static X25519 keypair, which the WhatsApp app
creates once per installation and stores in the protected `keystore`
storage. The keypair plays three roles: (a) the `authkey` parameter in
the `/v2/code` and `/v2/register` HTTP requests; (b) the source of the
AES key for integrity blobs (`gpia`, `_gi`, `_gg`, `integrity_payload`,
`jws`); (c) the client static key in the chat socket Noise handshake, by
which the server identifies the account on login.

## Place in the flow

1. First installation: `AnonymousClass139` (AuthKeyStore, DI id 6015)
   generates the `C1Jp.A00()` keypair and persists it in the `keystore`
   SharedPreferences (see the "Storage" section below).
2. Registration: the `POST /v2/code` and `POST /v2/register` requests.
   All 7 signal parameters are added by `X/C28804Cju.java`
   (`RegistrationKeyBundleHelper/addKeyBundleParams`, lines 11–43),
   called from `com/whatsapp/registration/core/http/KotlinRegistrationBridge.java`
   (method `A0U`, calls at lines 107, 237, 352, 470, 564, 730, 825). The
   `authkey` value is taken in a single line:
   `((AnonymousClass139) ...).A0D().A02.A01` — the keypair's public 32
   bytes.
3. Account login: the same public key goes (already encrypted) into the
   Noise handshake — `ClientFinish.static` in full XX or
   `ClientHello.static` on resume (see below), while the private half is
   used locally for DH.
4. After login: the server can rotate the keypair via the `w:auth:key`
   IQ (`X/AnonymousClass185.java`), in which case the `authkey` value
   changes in subsequent requests/logins.

## Wire format (+live example)

- Parameter value: `base64.RawURL(pub[32])` — unpadded base64url of
  exactly 32 bytes of the X25519 public key. The string length is always
  **43 characters** (256 bits / 6 = 42.(6) → 43). Characters:
  `A-Z a-z 0-9 - _`, no `=`.
- A single shared builder method does the encoding: `C40787I4g.A04(name,
  bytes)` → `android.util.Base64.encodeToString(bytes, 11)`, flags
  `11 = 8|2|1` = `URL_SAFE | NO_WRAP | NO_PADDING` (identical to
  `base64.RawURLEncoding`).
- The 0x05 prefix is NOT added — unlike `e_ident`/`e_skey_val`, where
  the prefix is merely stripped before sending, here it never existed at
  all: the public key constructor `C27011Jk` requires `length == 32` and
  fails with `IllegalArgumentException("Wrong length: N")` on any other
  size.

Live values (two independent sessions, Samsung SM-A325F phone):

| Session | WA version | authkey (43 chars) |
|---|---|---|
| 2026-09-22, `/v2/code` + `/v2/register` | 2.26.35.75 (263507522) | `rsgyL8y0zyOXIJT4KofOJGJaxrlKUp6TuBsPvGzZAzs` |
| 2026-09-28, `регистрация.txt` | 2.26.37.73 (263707322) | `Me_A3cdy1KTFeg8dfS-f27vIVDFeRMqXhOJMifg-b2w` |

In both captures the `authkey` value is byte-for-byte IDENTICAL in
`/v2/code` and `/v2/register` of the same installation — the parameter
is not recreated between requests.

## How it is formed in the app

### Keypair generation

`X/C1Jp.java:16-20`:

```java
public static C1Jp A00() {
    C1KL c1kl = C1KK.A00("best").A00;
    byte[] bArrGeneratePrivateKey = c1kl.generatePrivateKey();
    return new C1Jp(new C27061Jr(bArrGeneratePrivateKey),
                    new C27011Jk(c1kl.generatePublicKey(bArrGeneratePrivateKey)));
}
```

- `C1KK.A00("best")` — the provider factory for
  `org.whispersystems.curve25519.*` (X/C1KK.java:8-34): `"best"` →
  `OpportunisticCurve25519Provider` (native libsignal-native, fallback
  `JavaCurve25519Provider`).
- `generatePrivateKey()` — 32 random bytes + clamp: `b[0] &= 248`,
  `b[31] &= 127`, `b[31] |= 64` (JavaCurve25519Provider.java:410-421).
- `generatePublicKey()` — X25519 ScalarBaseMult, exactly 32 bytes.
- Invariant: `C27011Jk` (public, X/C27011Jk.java:16-23) requires
  `length == 32`; `C27061Jr` is the private half.
- The keypair plaintext when serialized is 64 bytes `priv32 ‖ pub32`
  (`C1Jp.A01()` → `AbstractC27051Jq.A06`; parsing —
  `AbstractC27051Jq.A07`).

### Entry point during registration

`X/C28804Cju.java:16` — the store's `A0D()` returns the keypair,
`.A02.A01` is the public 32 bytes; they are then placed into
`C28498Cee` (KeyBundleData, field `A00 = authKey`,
X/C28498Cee.java:14) and added as the `authkey` parameter (line 27).
`AnonymousClass139.A0D()` (X/AnonymousClass139.java:425-434) throws
`RuntimeException("AuthKeyStore/failed to get client static key pair")`
if the keypair is missing.

### STORAGE (where it is persisted)

File `/data/data/com.whatsapp/shared_prefs/keystore.xml`, two
semantically mutually exclusive keys:

| `keystore` pref | JSON array format | Encryption | Strength |
|---|---|---|---|
| `client_static_keypair_enc` | `[0, b64(ct), b64(iv)]` | AES-256-GCM, key `aes_auth_key` in AndroidKeyStore (TEE) | the blob is useless off the device |
| `client_static_keypair_pwd_enc` | `[2, b64(ct), b64(iv16), b64(salt4), b64(rand16)]` | AES-128-OFB, key = PBKDF2-HMAC-SHA1(16 iterations) of the APK constant `C0UA.A0X` + rand16 from the blob | decryptable offline |

The plaintext of both blobs is the same 64 bytes `priv32 ‖ pub32`. The
decision of which blob to write is made by `AnonymousClass139.A03(z)`
(generation: first a KeyStore attempt `A0B`, on failure — the pwd
`A05`; pwd write — lines 508-512). The decrypted keypair is cached
forever in the singleton field `A00` as raw `byte[]` without zeroization
(AnonymousClass139.java:16, 272-273). Full reset — `A0E()` (lines
436-452): removes both pref keys with the log
`"clearing client static key pair"`, after which the keypair is
regenerated.

The weak/strong storage choice is controlled by: the vendor blacklist
(AB flag 388 — then AndroidKeyStore is disabled entirely, lines
356-365), the "verifying stage" (AB flags 831/378 — both copies are
stored in parallel), the failure counter
`can_user_android_key_store=false` and the
`"failed too much must recover"` path.

### ROLE (b): the derived AES key of integrity blobs

The AES key of the envelope of all registration and login integrity
blobs = `SHA256(STANDARD_base64(authkey_raw_32B))` — that is, of the
base64 WITH padding and `+/` produced from the same 32 bytes. Live
derivations:

| Session | authkey | SHA256(std_b64(authkey)) |
|---|---|---|
| 2.26.35.75 | `rsgyL8y0…Azs` | `R41B51Od02uXyjkarGGxbrDvQghVHzMkl9W+HOQDJK4=` |
| 2.26.37.73 | `Me_A3cdy1KTFeg8dfS-f27vIVDFeRMqXhOJMifg-b2w` | `pjhfD73u7ZYPErnsKS6NCcqYU/rzGIYKsh5vE0Uw5qo=` |

With this key (AES-256-CBC, IV||ct, PKCS7) the `gpia`, `_gi`, `_gg`
blobs were cleanly decrypted in the first session, and in the second —
the same ones plus `integrity_payload` and `jws` (five blobs of one
installation). The envelope nonce is the unpadded base64url of the same
32 bytes, i.e. the authkey string value itself.

### ROLE (c): the Noise handshake client static key

- Full handshake (first login, no saved server static): after verifying
  the server certificate, the client encrypts its static public key
  (`encrypt_cs`) and places it in `ClientFinish.static`, then DH se
  (`ecdh_se`); the login payload — separately in `ClientFinish.payload`
  (X/C1KE.java, span `send_client_finish`; analysis — chapter 10,
  section "ClientFinish — this is where login happens"). The server
  recognizes the account by the pair "username in the payload + this
  static".
- Resume (server static saved, classic IK): the encrypted static is
  placed directly in `ClientHello.static`; in PQ mode the `static` and
  `ClientPayload` fields are concatenated and encrypted as a single
  blob in `payload`.
- If authkey cannot be read — the client itself aborts the handshake
  with a type 8 login error (failure analysis in X/C0SW.java).
- The private half does not leave the handshake: it is used locally for
  DH and passed to the native layer (see the next section).

## Native implementation

- Generation and all Curve25519 operations go through
  `C1KK.A00("best")` = `OpportunisticCurve25519Provider`: priority is
  the native provider from libsignal-native (shipped inside libs.so),
  the fallback is pure Java (`JavaCurve25519Provider`,
  org/whispersystems/curve25519/).
- The private half of the keypair is passed to the wamsys native layer:
  `JniBridge` (`com/whatsapp/wamsys/JniBridge.java`, case 3, lines
  ~476-484) reads `((AnonymousClass139) C00C.A02(6015)).A0D().A01.A01`
  (the private 32 bytes), checks `length == 32` and on error writes
  `"AuthKeyStoreImpl/the key length is not expected/privateLength=N"` —
  the native stack (mns/socket) uses the key for protocol processing.
- The Noise handshake itself (mixHash/encrypt DH operations) is in the
  Java classes `X/C1KX`/`X/C1KE`; native receives the already-ready
  keys via callbacks.

## Value selection conditions

- The value is stable for the entire lifetime of the installation: one
  keypair per installation, all registration/login requests before
  rotation carry the same authkey.
- Regeneration happens ONLY: (1) full reset `A0E()` — clearing
  prefs/resetting the KeyStore on failures or data deletion; (2)
  server-side rotation IQ `w:auth:key`
  (`X/AnonymousClass185.java:12-41`): sent at most once per day
  (`>= 86400000` ms since
  `last_succeeded_auth_key_rotation_attempt`), when
  `remaining_auth_key_rotation_attempts > 0`, with a 30-minute backoff
  on failures (`1800000` ms); the new `C1Jp.A00()` keypair is sent in
  the `<key>` node (public 32 bytes, `C0P0.A04(bArr, 32L, 32L)` —
  length check), the response is processed by `X/C41722Ifn.java` and
  the keypair is written to storage.
- The format is invariant: 32 bytes → 43 unpadded base64url characters;
  a leading 0x05 byte is NEVER present in the value (unlike the storage
  serialization of identity/skey).

## Cryptography and encoding

- Keypair algorithm: X25519 (Curve25519, DJB type). Private key — 32
  bytes with standard clamping; public — 32 bytes little-endian
  (u-coordinate).
- Wire encoding: `Base64.encodeToString(bytes, 11)` =
  URL_SAFE|NO_WRAP|NO_PADDING; 32 bytes → 43 characters, no `=`.
  Standard base64 (with `+/=`) of the same bytes is used only inside
  the AES key derivation: `SHA256(std_b64(authkey))`.
- Storage encoding: the 64 bytes `priv‖pub` are encrypted as a whole
  (GCM or OFB+PBKDF2, see the table above); blob parsing —
  `X/C14810lj.java`, `X/C14820lk.java` (type 0 = KeyStore, type 2 =
  password).

## Examples

- Live `/v2/code` URL (fragment, wire order preserved):
  `…e_skey_val=…&authkey=rsgyL8y0zyOXIJT4KofOJGJaxrlKUp6TuBsPvGzZAzs&e_keytype=BQ&e_regid=extZ9w…`
  — the same `authkey` value in `/v2/register` of the same session.
- Decoding the live value: `rsgyL8y0…Azs` (43 chars) → exactly 32
  X25519 public bytes; `Me_A3cdy1KTFeg8dfS-f27vIVDFeRMqXhOJMifg-b2w`
  (43 chars) → exactly 32 bytes, the `-` characters confirm the
  URL-safe alphabet.
- Envelope derivation: `SHA256("rsgyL8y0…Azs" in std-base64) =
  R41B51Od02uXyjkarGGxbrDvQghVHzMkl9W+HOQDJK4=` — decrypts
  `_gg`/`gpia`/`_gi` from the live dump `whatsapp_v2_code (2).txt`.
- Rotation: IQ `<iq to xmlns='w:auth:key' type='set'><key>` with the new
  32 bytes (AnonymousClass185.java:26-33) — after confirmation the
  keypair in `keystore.xml` is replaced, and all subsequent logins use
  the new static.
