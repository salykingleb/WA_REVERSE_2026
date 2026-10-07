# fdid — installation phone ID, a UUID v4 string

## Place in the flow

`fdid` (phone ID / foreground device id) — an identifier of the app INSTALLATION (not of the
number, not of the session). Generated once on the first access, stored forever (until the data
is cleared), sent as a plain UUID string in `/v2/code`, `/v2/register`, funnel-log, pre-PN logs
and account-defence requests. The call point — `IF6.A0C(if6)` (X/IF6.java:1086-1088):
```java
public static String A0C(IF6 if6) {
    return A04(if6).Aso().A01;    // IRQ → InterfaceC26881Ig (C1Ih) → C1Ij.A01 = строка UUID
}
```
called in `RequestCodeRepository$requestCode$2.java:352/411` and passed as the 4th argument into
`KotlinRegistrationBridge.A0S(builder, lg, lc, fdid, expid)` → `C40787I4g.A01("fdid", str3)`
(KotlinRegistrationBridge.java:1071-1076).

## Wire format (+LIVE example from the capture)

A plain UUID v4 string, lowercase, with hyphens, 36 characters — WITHOUT any transformations
(the A01 method puts it as is; percent-encoding will be performed by the native side, but for
UUID characters there is nothing to encode — `[0-9a-f-]` are all unreserved).

LIVE example (Samsung SM-A325F, capture of 2026-09-22):
```
fdid=da1e6667-b5ed-4fa6-9828-53283393c119
```
— version 4 (the version nibble `4` in the third group: `4fa6`), RFC 4122 variant (`9` at the
start of the fourth group: `9828` → 10xx). The value is IDENTICAL in `/v2/code` and
`/v2/register`.

## How it is formed in the app

**Storage/generator — `X/C1Ih.java` (PhoneIdStore, implements InterfaceC26881Ig):**
```java
public synchronized C1Ij Aso() {
    String strA0e = c018208l.A0e();                       // чтение SharedPreferences "phoneid_id"
    long jA0B = c018208l.A0B("phoneid_timestamp");        // и "phoneid_timestamp"
    if (strA0e == null || jA0B == -1) {                   // нет → сгенерировать
        String string = UUID.randomUUID().toString();     // UUID v4
        c1Ij = new C1Ij(string, C08A.A00(this.A00));      // + текущее время
        CR5(c1Ij);                                        // сохранить
    } else {
        c1Ij = new C1Ij(strA0e, jA0B);
    }
    return c1Ij;
}
public synchronized void CR5(C1Ij c1Ij) {
    c1Ii.A01().putString("phoneid_id", c1Ij.A01).apply();
    c018208l.A0z("phoneid_timestamp", c1Ij.A00);
}
```
- `UUID.randomUUID()` — the standard Java UUID v4: 122 random bits, version 4, RFC 4122 variant.
- **Persistence:** the pair of SharedPreferences keys `phoneid_id` (the UUID string) and
  `phoneid_timestamp` (the generation time, long). Written once per installation
  (`synchronized`, repeated calls read from SP).
- **When regenerated:** only if both/any of the keys is missing (`strA0e == null ||
  jA0B == -1`) — i.e. after clearing the app data or on a new installation. Between requests,
  between numbers, between APK reinstalls over the existing data — unchanged.

## Native implementation

Not involved: the value belongs to the Java side, the string goes into the native query builder
as a ready value via `A01` (not included in the "already encoded" Set — the native side encodes
it; for a UUID this is a no-op). The name `"fdid"` is present in the native rodata table of
registration parameters (libwhatsapp.so).

## Value selection conditions (variants, ranges, when absent)

- Always a UUID of **version 4**, RFC 4122 variant: `xxxxxxxx-xxxx-4xxx-yxxx-xxxxxxxxxxxx`,
  where y ∈ {8,9,a,b}; lowercase hex.
- The parameter is sent ALWAYS (there is no null branch) in all the listed requests.
- The value does not depend on the phone number or the session — only on the installation.

## Cryptography and encoding

- Generation: `UUID.randomUUID()` — internally the JDK's `SecureRandom` (not `getInstanceStrong`
  directly, but the platform's cryptographically strong PRNG). No other cryptography.
- Encoding: none. The 36-character string `[0-9a-f-]` requires no percent-encoding.

## Examples

- Live value: `da1e6667-b5ed-4fa6-9828-53283393c119` (v4, capture 2026-09-22).
- Version breakdown: group 3 = `4fa6` → version nibble 4; group 4 = `9828` → binary variant 10xx
  (RFC 4122).
- On the wire in the code request (fdid is 49th of 50), in register — 47th of 49 (the native
  order, live capture).
