# id — random identifier of a code request attempt, persistent per phone number

## Place in the flow

`id` — 20 random bytes that the app generates on the first code request for a number and
**reuses for all repeated attempts for the same number** (the same logic as backup_token,
but the value is independent and stored in a different place — in the file `files/rc2`, not in
SharedPreferences). It participates in `/v2/code`, `/v2/register`, as well as in standalone code
requests (requestCodeForStandaloneVerification) and funnel-log. The generator call —
`RequestCodeRepository$requestCode$2.java:342`: `byte[] bArrA0u = if6.A0u(str2, str3)` (str2=cc,
str3=in). It gets into the builder via `KotlinRegistrationBridge.A0T(builder, asi, bArrA0u, bArrA0t)`
→ `C40787I4g.A05("id", bArr)` (KotlinRegistrationBridge.java:1078-1084).

## Wire format (+LIVE example from the capture)

A percent-encoded string (RFC3986, `%XX` UPPERCASE) of 20 raw bytes. The on-the-wire length ranges
from 20 characters (all bytes fall into the unreserved `A-Za-z0-9-._~`) to 60 (all bytes are "bad",
3 characters per byte). Live capture (Samsung SM-A325F, cc=7, in=9206309125): `id` is present and
**IDENTICAL** in `/v2/code` and `/v2/register` (a line from the capture: `id=Wp1P...` — a 20..60
character percent-string). Position on the wire in the code request: the 5th key after
`method, backup_token, _gs, sim_mnc`.

## How it is formed in the app

**Generator of the 20 bytes — `C00L.A0G()` (X/C00L.java:525-533):**
```java
public static byte[] A0G() {
    try {
        KeyGenerator keyGenerator = KeyGenerator.getInstance("AES");
        keyGenerator.init(160, AbstractC42871v4.A00());
        return keyGenerator.generateKey().getEncoded();
    } catch (Exception e) { throw new RuntimeException(e); }
}
```
- `KeyGenerator "AES"` with explicit initialization to **160 bits** = exactly 20 bytes at the
  output (`getEncoded()` returns the raw key bytes).
- The entropy source is `AbstractC42871v4.A00()` (X/AbstractC42871v4.java): on SDK ≥ 26
  `SecureRandom.getInstanceStrong()`, below / when the provider is absent — `new SecureRandom()`.

**Read/write with persistence per number — `IF6.A0u(String cc, String in)` (X/IF6.java:1246-1256):**
```java
public byte[] A0u(String str, String str2) {
    String strA00 = AbstractC39008HQk.A00(AbstractC49602Fy.A0V(str, str2)); // хеш от cc+in
    Application application = this.A03;
    byte[] bArrA0I = C00L.A0I(application, strA00);   // чтение из files/rc2 по ключу-хешу
    if (bArrA0I != null) return bArrA0I;              // промах → новая генерация
    byte[] bArrA0G = C00L.A0G();
    C00L.A09(application, strA00, bArrA0G);           // сохранить под ключом-хешем
    return bArrA0G;
}
```
- Storage key: `AbstractC39008HQk.A00(cc + in)` — a hash of the concatenation of the country code
  and the national number (for the capture: `"7" + "9206309125"` = `"79206309125"`).
- A single file `files/rc2` can hold several records — one per number (the hash key in the name).

**Persistence: the file `files/rc2` (X/C00L.java:206-238 A09 / 576-614 A0I).**
File format (Java serialization of `ObjectOutputStream` on top of its own bytes):
```
[2B магия A04={0,2}] [4B salt] [16B IV] [AES/OFB/NoPadding(20B id)]
```
- Encryption: `Cipher "AES/OFB/NoPadding"`, key — `SecretKeySpec(A0K(salt4B, keyNameString), "AES/OFB/NoPadding")`.
- Encryption key derivation `A0K` (X/C00L.java:657-667): `PBKDF2WithHmacSHA1And8BIT`, password =
  the bytes of the **key name** (the number hash) as char[], salt = 4 random bytes,
  **16 iterations, 128 bits**. That is, the AES key is derived from the NAME of the storage key
  (the number hash) — the id itself does not participate in the key derivation.
- Read `A0I`: magic check (otherwise `C001400q` "recovery token header mismatch" → the file is
  deleted, returns null → regeneration), length check ≥ 42 (2+4+16+20), OFB decryption.
- The `rc2` file exists, and in the live capture the flag `hasinrc=1` is visible
  (`GCN.A1W(filesDir, "rc2")`, X/IF6.java:1154).

**When regenerated:** only if `rc2` for the given number is missing/corrupted. Between code request
attempts of one number, between `/v2/code` and `/v2/register`, between app restarts — the same
value. A new number → a new hash key → independent 20 bytes.

## Native implementation

The value itself is entirely Java. The parameter name `"id"` is present in the native table of
registration parameter names (rodata libwhatsapp.so @0x2f35e0+). The native side receives from
Java an ALREADY percent-encoded string (the key is in the "already encoded" Set of
`C40787I4g.A05`) and inserts it into the query without re-encoding.

## Value selection conditions (variants, ranges, when absent)

- The raw material is always exactly 20 bytes (AES-160 KeyGenerator); the probability of "pretty"
  values is negligible.
- Wire string length range: 20 (all unreserved) … 60 (all `%XX`); the expected value ≈ 33-36
  characters (each byte is unreserved with probability 66/256 ≈ 26%).
- The parameter is sent ALWAYS (there is no null branch): both in code, and in register, and in
  standalone.
- The absence of `files/rc2` does not cancel the parameter — a new value is simply generated.

## Cryptography and encoding

- Generation: AES KeyGenerator 160 bits over `SecureRandom.getInstanceStrong()` (SDK≥26) —
  cryptographically strong 20 bytes.
- Storage: AES-128/OFB (without padding, length multiple of 1), key PBKDF2-HMAC-SHA1
  (16 iterations) from the key name; salt 4B and IV 16B random on every write
  (A0H → AbstractC42871v4.A00()).
- Encoding into the request: `HSJ.A00(bytes)` — RFC3986 percent-encode, unreserved
  `A-Za-z0-9-._~` as is, the rest `%XX` UPPERCASE (X/HSJ.java:8-46, alphabet X/C40787I4g.A02).

## Examples

- Wire length formula: for 20 bytes B_i the string = Σ(len(B_i ∈ unreserved ? 1 : 3)).
- Structure example: bytes `4E 6F 2F 00 …` → `No%2F%00…` (0x2F='/' and 0x00 escaped UPPERCASE).
- Live capture: id is the same in `/v2/code` and `/v2/register` (live capture, the line "id/…
  IDENTICAL in code and register"), hasinrc=1 confirms the presence of rc2 on the device.
