# Additional conditional /v2/register parameters: fid, preloads_*, cred_token, old_phone_number, context, security_code (2FA)

All the parameters below are optional: in the live consumer flow (a fresh
registration, mode=0) NOT ONE of them is sent. Each is enabled by its own
condition.

---

## 1. fid — Firebase Installations ID

### Position in the flow

A conditional parameter of the verify-flow core (IF6.A0W), added to the map
of all registration requests (register/code/exist/consent/security) if the
device has a saved Firebase Installations identifier.

### Format (+live example)

The string `"<creationTimestamp>|<name>"` (`%d|%s`):
- creationTimestamp — epoch-millis of the installation's creation (22
  digits, long);
- name — the 22-character Firebase Installation ID (Base64url, ending with
  ":API-level", e.g. `…:O9`).

An example of the form:
`fid=1726959242000|fKx8vAbCdEfGhIjKlMnOpQ:O9`. Live consumer dump: absent.

### How it is formed in the app (class file:line, code)

```java
// X/IF6.java:616-622
public static final void A0W(IF6 if6, java.util.Map map) {
    C1Ij c1Ij = ((IRQ) C05D.A03(if6.A0H)).Aso();
    if (c1Ij != null) {
        map.put("fid", B0W.A1Z(AbstractC49542Fs.A1F("|",
                AnonymousC00000.A0A(c1Ij.A00),     // long ts → String
                AbstractC49512Fp.A01(c1Ij.A01)))); // name
    }
}

// X/IRQ.java:10-18 — the source: prefs, not live Firebase:
public synchronized C1Ij Aso() {
    String name = prefs("phoneyid_id");
    long ts     = prefs("phoneyid_timestamp");
    if (name == null || name.empty() || ts == -1) return null;   // → the field is not put
    return new C1Ij(name, ts);
}
```

`phoneyid_id`/`phoneyid_timestamp` are populated by the Firebase
Installations SDK (`FirebaseInstallations.getId()`) and cached in prefs (the
IRQ wrapper: CR5 — write, Aso — read).

### Value selection conditions

Sent ⇔ prefs contain a non-empty phoneyid_id AND ts != -1, i.e. the GMS
device has received an Installation-ID at least once (usually right after
the first start with Play Services). On devices without GMS / on a Firebase
failure — the field is absent. Consistent with has_play_store: false →
no fid.

### Cryptography and encoding

None; the `%d|%s` string → UTF-8 → percent-encoding (the colon and `|` are
RFC3986 safe characters and are not escaped; in form-urlencoded, `|` is left
as is by a number of parsers).

---

## 2. preloads_app_manager_id / preloads_attribution — preinstalled firmware

### Position in the flow

A pair of conditional IF6.A0U parameters: attribution of a device on which
WhatsApp was preinstalled by the manufacturer/carrier (preload channel)
rather than installed from the Play Store.

### Format (+live example)

- `preloads_app_manager_id` — a string identifier of the preload manager
  (a value from the system ContentProvider, saved in prefs);
- `preloads_attribution` — a JSON attribution string (`attribution_json`,
  the `preloads_payout_attribution_json` field).

Live dump (Play channel): both absent.

### How it is formed in the app (class file:line, code)

```java
// X/IF6.java:580-592
ICH ich = ...;
String managerId  = ich.A04(application);   // prefs "preloads_app_manager_id"
if (managerId != null)  map.put("preloads_app_manager_id", ...);
String attribution = ich.A05(application);  // prefs "preloads_payout_attribution_json"
if (attribution != null) map.put("preloads_attribution", ...);
```

Populating the prefs — X/ICH.java: the DevicePolicy/PreloadManager
ContentProvider is queried for `attribution_json` (cursor, :56–63) and it is
stored in prefs (:63, :87 `preloads_app_manager_id`; :102–104
`preloads_payout_attribution_json`). The read is cached one-time in the
A01/A02 fields (:109–118).

### Value selection conditions

Present only on firmware with a preload contract (OEM/carrier builds),
where the system ContentProvider returns the attribution. On ordinary Play
Store installs both are null → the fields are not sent.

### Cryptography and encoding

None; strings → UTF-8 → percent-encoding.

---

## 3. cred_token — token for passkey-disabled 2FA (number change)

### Position in the flow

A conditional IF6.A0M parameter (called both from the register verify flow
and from exist, consent): a server token issued for a specific number when
the passkey is disabled and a 2FA repeat is required on
re-registration/number change.

### Format (+live example)

An opaque string (a server value from the JSON store). Live dump: absent.

### How it is formed in the app (class file:line, code)

```java
// X/IF6.java:430-440
public static final void A0M(IF6 if6, String cc, String in, java.util.Map map) {
    if (((C0CS) C05D.A03(if6.A05)).A0w(25565)) {            // AB flag 25565
        I8V i8v = ...;
        String key = AbstractC49602Fy.A0V(cc, in);          // the "cc|in" key... (see below)
        String token = map_from_prefs.get(key);             // I8V.A01(i8v)
        if (token != null) map.put("cred_token", B0W.A1Z(token));
    }
}

// X/I8V.java:34-47 — the store: a JSON object in prefs
String json = prefs("passkey_disabled_cred_token_map");      // { "cc...number": "token" }
```

`AbstractC49562Fu.A0z(strA0V, I8V.A01(i8v))` — selecting the value by the
"full number" key (A0V — the cc+in concatenation, the same helper as in
old_phone_number).

### Value selection conditions

Sent ⇔ AB flag 25565 is active AND `passkey_disabled_cred_token_map` has an
entry for the number being registered (the server previously issued a token
in the "passkey disabled" scenario). A fresh registration — no map → the
field is absent.

### Cryptography and encoding

The token is generated by the server; the client stores it as is. Encoding:
string → UTF-8 → percent.

---

## 4. old_phone_number — the previous number (number change only)

### Position in the flow

A conditional IF6.A0Y parameter: the current (old) number of the active
account — sent when a NEW number is verified on top of an existing account
(change number / migration) so that the server can link the transfer.

### Format (+live example)

The string `"<cc><number>`" without a separator
(AbstractC49602Fy.A0V(cc, number)). Live dump (fresh registration): absent.

### How it is formed in the app (class file:line, code)

```java
// X/IF6.java:632-647
public static final void A0Y(IF6 if6, java.util.Map map, boolean z) {
    Me me = ((C017908i) wizardState.get()).Aq1();     // the current user
    if (me == null) {
        if (!z) return;                                // normal registration: exit
        me = wizard.A09().A0F;                         // z=true: load from the wizard
        if (me == null) return;
    }
    map.put("old_phone_number", B0W.A1Z(AbstractC49602Fy.A0V(me.cc, me.number)));
}
```

In the verify flow (register) A0Y is called with z=false: the field appears
only if the Me account is already initialized in memory (migration). In
exist (A0j:1411) — with the caller's boolean z (reinstall-check).

### Value selection conditions

Present ⇔ there is an active Me (the number is already registered on the
device) and a different number is being registered. A fresh install —
absent.

### Cryptography and encoding

None; cc+in as a string → UTF-8 → percent.

---

## 5. context — alternative flow marker (unban / web / invite)

### Position in the flow

A bridge parameter (`A02("context", str9)`, KotlinRegistrationBridge.java:1305)
from the `$context` argument of verify$2. For alternative registration
scenarios it marks where the OTP came from.

### Format (+live example)

One of the strings (the C02S code values in IF6.A0j:1505–1521):
`poll_2fa` | `twofac_dynamic` | `web_registration` | `invite_registration` |
`unban_registration`; for the consent request — `dob`.

Live dump: absent (normal registration).

### How it is formed in the app

Scenario determination — IF6.A0j (checkIfExists, X/IF6.java:1362–1370):
```java
if (prefs "server_invite_otp" valid && not consumed)   → context=invite_registration
else if (prefs "unban_otp" valid && AB HWG.A01)        → context=unban_registration
else if (prefs "web_registration_otp" valid)           → context=web_registration
else normal flow — context is NOT set
```
$context is then carried through VerifyCodeUseCase$verifyCode$1 → verify$2
→ the bridge. In A07 the field is null-guarded: null → not put.

### Value selection conditions

Present only in the unban/web/invite flow (obtaining the OTP from SMS
stubs/a portal). Normal registration — absent (the live dump confirms).

### Cryptography and encoding

None; ASCII string → A02 (null-guard) → form-urlencoded.

---

## 6. security_code and the 2FA flow (IF6.A0n, endpoint /v2/security)

### Position in the flow

`security_code` — a TWO-FACTOR AUTHENTICATION parameter (a 6-digit PIN or a
password), not an OTP. The OTP in register always goes in `code`; the names
are not interchangeable. The 2FA send is a separate IF6.A0n method
(reg_http_verify_security_code), called from the VerifyTwoFactorAuth
screen.

### How it is formed in the app (class file:line, code)

```java
// X/IF6.java:1651-1789 — A0n(c39796HkI, cc, in, code, method, loginHFR, ...):
...
if (AB flag 26215) {
    if (str5 /*login method*/.equals(HG7.A05.wireValue /* "password" */)) {
        // 2FA PASSWORD: encryption and sending in the code field
        str7 = J3Z(this, str3, null, 8);        // :1702 — see below
        if (str7 == null) QPL fail "PASSWORD_ENCRYPT_FAILED";
    } else {
        // 2FA PIN: plaintext in the additional security_code parameter
        map.put("security_code", A1b(str3));    // :1725
    }
    // Both paths → bridge A07 → POST /v2/register (!) with code=str7 (or empty)
    new KotlinRegistrationBridge$registerPhoneNumberBlocking$1(..., str7 /*code*/, ...);
} else {
    // the old path (flag 26215 off) → bridge A0C verifySecurityCodeBlocking
    // → POST /v2/security (HXH.A0F = AbstractC40561Hxd.A0H)
    A06(this, cc, in, str3, "", map, id, backup_token, null);
}
```

Password encryption — X/J3Z.java, case 8 (the call
`new J3Z(this, password, null, 8)` from A0n:1702):

```java
// X/J3Z.java:214-225 (case 8):
CanonicalPasswordService cps = (CanonicalPasswordService) C05D.A03(((IF6) this.A01).A06);
objA00 = cps.A04(password, cont, C0OY.A00, true);   // canonicalization + encryption
```

Canonicalization (CanonicalPasswordService) brings the password to a
unified form (trim, NFKC normalization taking full-width variants into
account), then encrypts it with the server public key from
`public_config_json` (J3Z case 3:153–157 reads the config; errors are
classified in A0n as keyOrServer/io/crypto — :1708–1722). The result is a
ciphertext string, and that is what goes into `code`.

The 2FA "method" values — the HG7 enum (X/HG7.java, wireValue):
`twofac_pin`(0), `password`(1), `email_otp`(2), `oauth_email`(3), `sms`(4),
`voice`(5), `flash`(6), `wipe_full`(7), `wipe_offline`(8).

### Format (+live example)

- PIN (twofac_pin / email_otp / …): `security_code=123456` (plaintext in
  TLS);
- password: an encrypted blob in `code`, the security_code field is absent;
- the reset flow (A0m, resetSecurityCode): with flag 26215 — the same
  verifySecurityCodeViaRegister → /v2/register with `reset` and
  `wipe_token` in the map (IF6.java:820–823).

Live dump (no 2FA): both fields absent.

### Value selection conditions / endpoints

| Scenario | Endpoint | Field with the secret |
|---|---|---|
| OTP verification (our main flow) | /v2/register | code=OTP |
| 2FA PIN, flag 26215 ON (current) | /v2/register | security_code=PIN |
| 2FA password, flag 26215 ON | /v2/register | code=JWE-like password ciphertext |
| Any 2FA, flag 26215 OFF (legacy) | /v2/security (HXH.A0F) | inside the old verifySecurityCode |
| 2FA reset (reset) | /v2/register (+reset, wipe_token) | — |

### Cryptography and encoding

- PIN — plaintext; password — hybrid encryption with the server public key
  (public_config_json) over the canonicalized string; encryption failure =
  refusal to send (PASSWORD_ENCRYPT_FAILED).
- The additional map of the 2FA request is assembled by the same
  A0Q/A0Y/A0O/A0V/A0X/A0T/A0M/A0S (IF6.java:1691–1698), i.e. the parameter
  set repeats register's.

---

## Presence summary table (live consumer dump 2026-09-22, RU, mode=0)

| Parameter | In the request | Reason |
|---|---|---|
| fid | no | no Firebase Installation in prefs (or not read) — conditional |
| preloads_app_manager_id / preloads_attribution | no | Play install channel, the preload provider stays silent |
| cred_token | no | flag 25565/token map is empty |
| old_phone_number | no | fresh registration, Me=null, z=false |
| context | no | not unban/web/invite |
| security_code | no | 2FA was not requested |
| method / tos_version / vname | no | see the respective chapters |
