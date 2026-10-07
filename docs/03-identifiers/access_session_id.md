# access_session_id — registration session UUID v4 → 16 bytes → base64 RawURL

## Place in the flow

`access_session_id` — an identifier of the registration SESSION (the period from opening the
number-entry screen to the completion of verification). The same in all the session's requests
(`/v2/code`, `/v2/register`, funnel-log, pre-PN logs, account-defence). The call point —
`IF6.A0D(if6)` (X/IF6.java:1090-1092): `return A03(if6).A01();` → `C12040h1.A01()`. Called in
`RequestCodeRepository$requestCode$2.java:355/413`, passed into
`KotlinRegistrationBridge.A0T(builder, str5, bArr, bArr2)` → only if str != null:
`C40787I4g.A03("access_session_id", str)` (KotlinRegistrationBridge.java:1078-1084).

## Wire format (+LIVE example from the capture)

A UUID v4 string → 16 raw bytes BE → URL-safe base64 without padding: 22 characters, no `=`.
The version (4) and variant (RFC 4122) bits are preserved in the bytes.

LIVE example (Samsung SM-A325F, 2026-09-22): `access_session_id=22b9ef7800-3c46…` — 16 bytes in
b64url (a live value; the full 22-character RawURL encoding). IDENTICAL in `/v2/code` and
`/v2/register`. Position on the wire (the native code order): between `prefer_sms_over_flash`
and `sim_mcc`.

## How it is formed in the app

**Store/generator — `X/C12040h1.java`:**
```java
public final String A01() {
    String strA08 = ((C02900De) this.A00.A00.get()).A08();   // SharedPreferences "access_session_id"
    return strA08.length() == 0 ? A00() : strA08;            // пусто → сгенерировать
}
private final String A00() {
    String string = UUID.randomUUID().toString();            // UUID v4
    Log.i("AccessSession/generateUUID/..." + AbstractC42301u7.A12(string, 4));  // первые 4 символа
    SharedPreferences.Editor editorEdit = ((C02900De) this.A00.A00.get()).Ap6().edit();
    editorEdit.putString("access_session_id", string);       // сохранить
    editorEdit.apply();
    return string;
}
```
- SP read: `C02900De.A08()` (X/C02900De.java:458-461) — `getString("access_session_id", "")`,
  null → `""`.
- **Persistence:** SharedPreferences `access_session_id` (a UUID v4 string with hyphens).
  Generated on the first access, then reused. It lives until a reset/explicit rotation.

**Synchronization with the native side — `C12040h1.A02()` (reset):**
```java
public final void A02() {
    Log.i("AccessSession/resetSessionId");
    A00();                                     // новая генерация + запись в SP
    // если wamsys загружен (C13K.A00):
    JniBridge.jvidispatchIO(9, A01());         // протолкнуть UUID в нативный движок (op 9)
    // иначе: "WaMsysSetup/updateAccessSessionId/failed to update accessSessionId, not bootstrapped for reg"
}
```
The same UUID, on every generation, is synchronously sent into the native wamsys — so that the
Java map and the native session parameters stay consistent. A session reset (`A02`) is the value
rotation point.

**When regenerated:** a call to `A02()` (resetSessionId — a new registration session, for
example a number change / flow reset); an empty/missing value in SP. It does NOT change between
the code and register of one session — confirmed by a live capture.

## Native implementation

A Java-store value, but with mirroring into the native engine via `JniBridge.jvidispatchIO(9,
uuid)` (X/C12040h1.java:29) — the native side keeps a copy for its own session attribution. It
is put into the builder via `C40787I4g.A03` (UUID string → 16 bytes → RawURL-base64). The name
`"access_session_id"` is in the native rodata table of registration parameters.

## Value selection conditions (variants, ranges, when absent)

- Always a UUID of **version 4**, RFC 4122 variant
  (`xxxxxxxx-xxxx-4xxx-yxxx-xxxxxxxxxxxx`, y∈{8,9,a,b}).
- The wire length is always exactly 22 characters (16 bytes → RawURL without padding).
- In `A0T` the key is put only if the argument is not null — in practice the UUID is always
  available (generated on the first read), therefore the parameter is present in all the flow's
  requests.
- Stable within a session; changes on resetSessionId / data clearing.

## Cryptography and encoding

- Generation: `UUID.randomUUID()` — the platform's cryptographically strong PRNG (122 random
  bits).
- Encoding: `C40787I4g.A03` → `ByteBuffer.allocate(16).putLong(msb)` + `GCM.A1Z(buf, lsb)`
  → `GCK.A0o` = `Base64.encodeToString(b, 11)` (URL_SAFE|NO_WRAP|NO_PADDING). The hyphens of the
  UUID string do NOT get into the bytes — that is only a textual representation.

## Examples

- Live: `access_session_id=22b9ef7800-3c46…` (16B b64url) — the same in code and register
  (live capture).
- Transformation: UUID `22b9ef78-00xx-4xxx-yxxx-…` → bytes msb=`22 b9 ef 78 00 …`, lsb=`…` →
  b64url 22 characters; the version nibble 4 is visible in byte 6 (the high nibble 4), the 10xx
  variant — in byte 8.
- Generation log: `AccessSession/generateUUID/…22b9` (the first 4 characters of the UUID in the
  log).
