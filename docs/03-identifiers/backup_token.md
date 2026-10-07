# backup_token — backup registration token, 20 bytes in SharedPreferences

## Place in the flow

`backup_token` — the second independent 20-byte identifier of the registration session. Unlike
`id`, it is NOT tied to the phone number: one value per app installation, it lives in the
SharedPreferences `token_used_during_reg` in base64. It is sent in `/v2/code`, `/v2/register`,
standalone code requests and funnel-log. The call point —
`RequestCodeRepository$requestCode$2.java:343`:
`byte[] bArrA0t = if6.A0t("requestCodeForStandaloneVerification")` (the string argument is for
logs only). It gets into the builder via `KotlinRegistrationBridge.A0T(...)` →
`C40787I4g.A05("backup_token", bArr2)` (KotlinRegistrationBridge.java:1078-1084).

## Wire format (+LIVE example from the capture)

A percent-encoded string (RFC3986, `%XX` UPPERCASE) of 20 raw bytes — the same wire form as
`id`: length 20..60 characters. In the live capture (Samsung SM-A325F, 2026-09-22) `backup_token`
is present in all the requests and **IDENTICAL** in `/v2/code` and `/v2/register`; in the code
request it goes as the 2nd key right after `method` (the native order:
`method, backup_token, _gs, …`).

## How it is formed in the app

**The generator — the same `C00L.A0G()` (X/C00L.java:525-533)** as for `id`:
```java
KeyGenerator keyGenerator = KeyGenerator.getInstance("AES");
keyGenerator.init(160, AbstractC42871v4.A00());   // 160 бит = 20 байт, SecureRandom.getInstanceStrong
return keyGenerator.generateKey().getEncoded();
```

**Read/write — `IF6.A0t(String logTag)` (X/IF6.java:1056-1070):**
```java
public byte[] A0t(String str) {
    C05D c05d = this.A0M;
    byte[] bArrA0t = ((C02900De) C05D.A03(c05d)).A0t();     // чтение SP "token_used_during_reg"
    if (bArrA0t.length != 0) return bArrA0t;
    // лог "RegistrationHttpManager/<tag>/no backup token read from shared preferences, generate a new one"
    byte[] bArrA0G = C00L.A0G();
    C000800h.A06(bArrA0G);
    ((C02900De) C05D.A03(c05d)).A0o(bArrA0G);              // запись в SP
    return bArrA0G;
}
```

**Storage — SharedPreferences `token_used_during_reg` (X/C02900De.java:57-70):**
```java
public final synchronized void A0o(byte[] bArr) {   // запись: base64 флаги 3 (NO_PADDING|NO_WRAP)? — см. ниже
    A01(this, "token_used_during_reg", bArr);       // putString(base64(bArr))
}
public final synchronized byte[] A0t() {            // чтение
    bArrDecode = Base64.decode(Ap6().getString("token_used_during_reg", /*""*/ Voip.REJECT_REASON_DECLINED), 3);
}
```
- The value in SP is a base64 string of 20 bytes; the read goes through `Base64.decode(..., 3)`
  (`NO_PADDING|NO_WRAP` — in the app's style), an empty string by default → length 0 →
  regeneration.
- The default value `Voip.REJECT_REASON_DECLINED` is an empty-string constant in the obfuscation.

**BlockStore migrations (X/IEn.java — BackupTokenUtils):** when a token from previous versions is
present, it is read from BlockStore/files: `IEn.A00(context, ...)` parses the byte[] blob as an
LRUCache — first as protobuf, on the `BTCP` header ({66,84,67,80}) with an unsuccessful parse
writes `BackupTokenUtils/convertByteArrayToLRUCache/proto_header_but_parse_failed`, then as Java
serialization (`ObjectInputStream` over `GCK.A0T(bArr)` — a decrypt stream). This is the migration
path of the per-number token history; on a new installation this branch does not participate —
simply `C00L.A0G()`.

**Persistence and regeneration:** the value lives from the installation's first registration until
the app data reset. Between `/v2/code` and `/v2/register` — the same; between repeated attempts —
the same; for ANOTHER number — THE SAME (does not depend on the number, unlike id).

## Native implementation

The value is entirely Java. The name `"backup_token"` is in the native rodata table of
registration parameter names (libwhatsapp.so @0x2f35e0+). Java passes an already percent-encoded
string (the key was added to the "already encoded" Set by the method `C40787I4g.A05`), the native
side inserts it into the query as is.

## Value selection conditions (variants, ranges, when absent)

- The raw material is exactly 20 bytes (`C00L.A0G()`, AES-160), independent of `id` (two separate
  generator calls, no correlation).
- Wire length 20..60 characters (the expected value ≈ 33-36), like id.
- The parameter is sent ALWAYS — both in code, and in register; there is no null branch (unlike
  advertising_id).
- The value does NOT change when the number changes (id changes, backup_token does not).

## Cryptography and encoding

- Generation: AES KeyGenerator 160 bits + `SecureRandom.getInstanceStrong()` (SDK≥26,
  X/AbstractC42871v4.java) — 20 strong random bytes.
- Storage: base64 in SharedPreferences (not encrypted — unlike id in rc2).
- Encoding: `HSJ.A00` — RFC3986 percent-encode, unreserved `A-Za-z0-9-._~` unchanged, the rest
  `%XX` UPPERCASE (alphabet `0123456789ABCDEF`, X/C40787I4g.java:47-51).

## Examples

- Live capture: `backup_token` — the 2nd key in the code request, the same as in the register
  request of the same session (live capture: "id/backup_token/… IDENTICAL in code and register").
- Encoding example: byte 0x2B ('+') → `%2B`, 0x2F ('/') → `%2F`, the letter 'N' → `N`.
- In the device's SP `token_used_during_reg = <base64 of 20 bytes>` is stored; after the app's
  "clear data" the SP is empty → IF6.A0t generates and saves a new value.
