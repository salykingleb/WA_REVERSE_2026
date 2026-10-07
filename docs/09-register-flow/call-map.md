# POST /v2/register (OTP verification flow) — complete map of calls and parameters

App: WhatsApp Android 2.26.35.75 (versionCode 263507522).

## Position in the flow

`/v2/register` — the final step of registration: the client sends the entered OTP
code (plus the e2e bundle and a large set of telemetry) and, on success, receives
the credentials. The flow order on the device (confirmed by a live session on
2026-09-22, SM-A325F):

```
RegisterPhone (number entry)
  → /v2/exist        (IF6.A0j, checkIfExists; on every "Next" with a full number)
  → /v2/code         (IF6.A0m → bridge A08 generateAuthCode, method=sms/voice/…)
  → VerifyPhoneNumber (OTP entry screen)
  → /v2/register     (the actual flow; code = the 6-digit OTP)
  → (conditionally) /v2/consent  (if the response is fail/reason=consent/pending=dob)
  → (conditionally) /v2/security (2FA flow, IF6.A0n)
```

The complete chain of OTP verification calls (Java → bridge → HTTP):

```
VerifyPhoneNumber.A5L(clientMetrics:H8a, code, cc, in, method, codeEntryMethod)
    com/whatsapp/registration/app/verifyphone/VerifyPhoneNumber.java:1342
  └> new VerifyCodeUseCase$verifyCode$1(bzp=null, H8d, h8a, code, method,
        cc, in, authCodeContext=A12(), authChallenge=null, context=null,
        codeEntryMethod, codeVerificationMode=this.A01)          (:1356)
      com/whatsapp/registration/app/verifyphone/usecase/VerifyCodeUseCase$verifyCode$1.java
  └> C40735I1n.A01(bzp, h8a, code, method, cc, in, authCodeContext,
        authChallenge, context, cont, codeVerificationMode, codeEntryMethod)
      X/C40735I1n.java:34  (VerifyCodeRepository)
  └> VerifyCodeRepository$verify$2 (the invokeSuspend coroutine; in the main jadx
      dump the body was not decompiled — restored from an instruction dump of
      jadx CLI --show-bad-code):
        map = IF6.A0I(bzp, if6, clientMetrics, mistyped_state, codeEntryMethod)
              ← mistyped, vname, client_metrics, entered, mcc/mnc/sim_mcc/sim_mnc,
                network_operator_name, sim_operator_name, has_play_store,
                then A0O (network_radio_type, simnum, hasinrc, pid, rc)
                and A0R (sim_type, airplane_mode_type, cellular_strength, roaming_type)
        IF6.A0Y(map, false)   // old_phone_number — number change only
        IF6.A0Q(map)          // device_ram
        IF6.A0V(map)          // gpia (Play Integrity, flag 3753 inside)
        IF6.A0P(map)          // 2 hidden Settings.Global parameters (AB flag 7999)
        if6.A0q(map)          // recaptcha (JSON with stage)
        IF6.A0W(map)          // fid ("%d|%s", conditional)
        IF6.A0U(map)          // preloads_app_manager_id / preloads_attribution
        IF6.A0X(map)          // tos_version="5" (conditional)
        IF6.A0M(cc, in, map)  // cred_token (flag 25565, number change)
        map["passkey_login_status"] = HFx(prefs "passkey_login_stage").wireToken
        // "context" — put only for the unban/web/invite flow
        IF6.A0T(map)          // entrypoint = "suma" | "create_paa"
        IF6.A0S(map)          // NATIVE: JniBridge.jvidispatchOOO(16, app, wajContext)
                              //   → Map<String, ByteArray> (_gs/_gp/_ga/_gi/_gg/db/aid/_ge…)
  └> KotlinRegistrationBridge$registerPhoneNumberBlocking$1(
        bridge, hfr=null, lg, lc, fdid, expid, accessSessionId, cc, in,
        code=OTP, authCodeContext=null, method=<r31, usually null>,
        advertisingId=A0p(cc,"register_entrypoint"),
        waTwoFaContactPoint=null, baseUrl, domainFrontingList, map,
        id(20 bytes), backup_token(20 bytes), authResponse=null)
  └> KotlinRegistrationBridge.A07  (core/http/KotlinRegistrationBridge.java:1277)
      → C40787I4g (RegistrationRequestBuilder) → RetryingHttpClient.A01
      → POST /v2/register  (line :1314, constant HXH.A0A)
```

Response parsing — also in A07 (`parseRegisterPhoneResponse`, :1346 onward):
`status=fail` + `reason` (incorrect / stale / too_many_guesses / consent /
security_code / blocked / mismatch / second_code / sms_required / …); with
`reason=consent` + `pending=dob`, the DOB flow `/v2/consent` is started.

## Format (+live example)

`application/x-www-form-urlencoded` (percent-encoding of values; some fields
use a dedicated encoder, see the encoding table below). Live dump of 2026-09-22
(RU, Wi-Fi without SIM), the actual field order on the wire (49 parameters):

```
code=111111
backup_token=<percent(20B)>  sim_mnc=000  _gs=<AES-blob>  id=<percent(20B)>
device_ram=3,56  db=<locale-dependent>  _gg=<AES-blob>  sim_operator_name=
passkey_login_status=not_attempted  lg=ru  rc=0  network_radio_type=1
pid=28104  _ge=<AES-blob>  cellular_strength=5  gpia=<AES-blob>  hasinrc=1
aid=<32B b64std>  network_operator_name=  mcc=000  _gp=<AES-blob>  mnc=000
entered=1  access_session_id=<b64url 16B>  sim_mcc=000  roaming_type=0
mistyped=7  airplane_mode_type=0  has_play_store=true  sim_type=0
e_ident=<b64url>  e_skey_sig=<b64url>  entrypoint=suma  in=9206309125
expid=<b64url 16B>  simnum=0  cc=7  e_skey_val=<b64url>  authkey=<b64url 43>
e_keytype=BQ  e_regid=<b64url>  _gi=<AES-blob>  client_metrics={"attempts":0,…}
e_skey_id=<b64url>  fdid=<UUID v4>  recaptcha={"stage":"ABPROP_DISABLED"}
_ga=<AES-blob>  lc=RU
```

Live dump facts: ABSENT from /v2/register — `method`, `tos_version`,
`advertising_id`, `vname`, `clicked_education_link` (the first three are
confirmed gating, see the respective chapters).

## How it is formed in the app (class file:line, code)

### Map core — IF6.A0I (X/IF6.java:340–385)

```java
public static final LinkedHashMap A0I(BZP bzp, IF6 if6, H8a h8a, String str, int i) {
    TelephonyManager tm = ...A0K();
    // mcc/mnc/sim_mcc/sim_mnc from getNetworkOperator()/getSimOperator() (A0K:411-428)
    if (str != null) map.put("mistyped", ...);                       // :364-366
    if (bzp != null) map.put("vname", Base64.encode(bzp.toByteArray(), 11)); // :367-372
    Log.i("RegistrationHttpManager/verifyCode/vname-attached|vname-absent");
    map.put("client_metrics", A1b(A1O(h8a.A01()), UTF8));            // :374-376
    map.put("entered", A1b(String.valueOf(i), UTF8));                // :377
    map.put("network_operator_name", ...);                           // :379 (empty if null)
    map.put("sim_operator_name", ...);                               // :380
    map.put("has_play_store", ...A1Y(C1Iw.A02(app, "com.android.vending"))...); // :381
    A0O(if6, map);   // network_radio_type, simnum, hasinrc, pid, rc  (:1139-1158)
    A0R(if6, map);   // sim_type, airplane_mode_type, cellular_strength, roaming_type
                     // — only under AB flag 4435 (:527-551)
}
```

### Request assembly — KotlinRegistrationBridge.A07 (:1277–1346)

```java
C40787I4g b = A01(objA01);
if (hfr != HFR.A03) A0R(b, cc, in);          // :1066  A01("cc"), A01("in")
A0S(b, lg, lc, fdid, expid);                 // :1071  A01 lg/lc/fdid, A03 expid
A0T(b, accessSessionId, id, backup_token);   // :1078  A03 access_session_id, A05 id/backup_token
b.A01("code", code);                         // :1301  the OTP as is
if (authResponse != null) b.A00.put("auth_response", GCK.A0o(...)); // :1302-1304
b.A02("context", str9);                      // :1305  null-guard
b.A02("method", str10);                      // :1306  null-guard (see README_code)
b.A02("advertising_id", str11);              // :1307
A0Q(b, hfr, login);                          // :1059  A02("login"), "type" when hfr!=null
A0U(b);  // C28804Cju.A00: authkey, e_ident, e_keytype, e_regid,
         // e_skey_id, e_skey_val, e_skey_sig — A04 (RawURL b64)   (X/C28804Cju.java:27-33)
b.A06(map);                                  // :1312  all additional verify-flow parameters
... RetryingHttpClient.A01(b, ..., HXH.A0A, ...)  // :1314  POST /v2/register
```

`id` and `backup_token` — 20 random bytes each (AES-KeyGenerator 160 bits,
X/C00L.A0G:525), persisted in `filesDir/rc2` keyed by the phone hash
(A0u:1246–1256); `hasinrc="1"` ⇔ the rc2 file exists (A0O:1154).

### "Which method adds which parameter" table

| IF6 method | Lines | Parameters | Condition |
|---|---|---|---|
| A0I (core) | 340–385 | mistyped, vname, client_metrics, entered, mcc, mnc, sim_mcc, sim_mnc, network_operator_name, sim_operator_name, has_play_store | always (mistyped/vname — when non-null) |
| A0O (inside A0I) | 1139–1158 | network_radio_type, simnum, hasinrc, pid, rc | always |
| A0R (inside A0I) | 527–551 | sim_type, airplane_mode_type, cellular_strength, roaming_type | AB flag 4435 |
| A0Y | 632–647 | old_phone_number | an active Me account exists (number change); in register it is called with z=false |
| A0Q | 511–521 | device_ram | always |
| A0V | 594–614 | gpia | AB flag 3753 and the presence of a Play Integrity token |
| A0P | 448–509 | 2 hidden Settings.Global parameters (names obfuscated as Integer[]) | AB flag 7999 |
| A0q | 975–1054 | recaptcha (JSON token/token_length/token_age/error/stage; null fields are removed) | always |
| A0W | 616–622 | fid = "%d\|%s" | prefs phoneyid_id/phoneyid_timestamp non-empty (Firebase Installations) |
| A0U | 580–592 | preloads_app_manager_id, preloads_attribution | firmware with a preload channel (prefs ICH) |
| A0X | 624–630 | tos_version="5" | AB flag 19561 AND eula_accepted_time>0 |
| A0M | 430–440 | cred_token | AB flag 25565 and an entry in passkey_disabled_cred_token_map |
| (verify$2) | — | passkey_login_status | always |
| (verify$2) | — | context | only the unban/web/invite flow |
| A0T | 569–578 | entrypoint="suma"/"create_paa" | suma — when the ToS version list is non-empty; create_paa — the PAA flow |
| A0S | 553–567 | the native set (_gs/_gp/_ga/_gi/_gg/db/aid/_ge…) | always (JNI) |
| bridge A07 | 1301–1311 | code, auth_response, context, method, advertising_id, login(+type) | see the chapters |

### X/AbstractC40561Hxd endpoints (strings obfuscated with XOR 0x12)

The decoder (`AbstractC40561Hxd.A00`, X/AbstractC40561Hxd.java:26–32):
`for c in s: out += chr(ord(c) ^ 18)`. Full decryption of the constants
(verified):

| Hxd constant | Obfuscated string | Endpoint | HXH alias | Used by |
|---|---|---|---|---|
| A0C | `=d =\`wu{afw\`` | **/v2/register** | HXH.A0A | bridge A07 (:1314) — OTP verification, 2FA password (flag 26215), resetSecurityCode (flag 26215) |
| A04 | `=d =q}\|aw\|f` | **/v2/consent** | HXH.A04 | bridge A06 (:1237) — DOB-consent |
| A0H | `=d =awqg\`{fk` | **/v2/security** | HXH.A0F | bridge A0C (:2436) — the old verifySecurityCode (without flag 26215) |
| A07 | `=d =q}vw` | **/v2/code** | HXH.A06 | bridge A08 generateAuthCode (:1594) — OTP request |
| A0E | `=d =wj{af` | /v2/exist | — | checkIfExists (IF6.A0j) |
| A0D | `=d =\`wuM}\|p}s\`vMspb\`}b` | /v2/reg_onboard_abprop | HXH.A0B | checkPreChatdABProps (IF6.A0l) |
| A02 | `=d =qzs~~w\|uw` | /v2/challenge | HXH.A02 | challengeRequest (IF6.A0h) |
| A0A | `=d =bsaaywkMsgfz` | /v2/passkey_auth | HXH.A08 | passkeyAuthResult (IF6.A0k) |
| A00 | `=d =sgf}q}\|t` | /v2/autoconf | HXH.A00 | autoconf flow |
| A01 | `=d =sgf}q}\|tMdw\`{t{w\`` | /v2/autoconf_verifier | HXH.A01 | autoconf-verify |
| A0G / A08 | `=d =sqdw\`{tk` / `…M\`wcgwaf` | /v2/acverify, /v2/acverify_request | HXH.A0E / HXH.A07 | carrier instant-verify |
| A03 / A0B | `=d =q~{w\|fM~}u` / `=d =b\`wMb\|Mq~{w\|fM~}u` | /v2/client_log, /v2/pre_pn_client_log | HXH.A03 / HXH.A09 | funnel client logs |
| A0F / A06 | `=d =vwd{qwM~}u}gfMaw\|v` / `…Mtwfqz` | /v2/device_logout_send, /v2/device_logout_fetch | HXH.A0D / HXH.A05 | device-logout |
| A0I | `=d =eta` | /v2/wfs | HXH.A0G | wfs |

Note: the earlier analysis map listed the "A0D→HXH.A0B" pair for /v2/code —
this is an error: HXH.A0B = Hxd.A0D = /v2/reg_onboard_abprop, while /v2/code
is Hxd.A07 → HXH.A06 (usage confirmed by the bridge, :1594).

### C40787I4g builder encoding table (RegistrationRequestBuilder)

| Method | Encoding | Parameters |
|---|---|---|
| A01 (name, string) | the string as is (C000800h.A0A check) | code, cc, in, lg, lc, fdid, dob, context |
| A02 (name, string) | string; **null → the field is not put** | method, advertising_id, login, security_code, context |
| A03 (name, UUID string) | UUID → 16 bytes (most+least significant bits) → GCK.A0o = Base64 flag 11 (URL_SAFE\|NO_WRAP\|NO_PADDING = RawURL) | expid, access_session_id |
| A04 (name, byte[]) | bytes → Base64 flag 11 (RawURL) | authkey, e_ident, e_keytype, e_regid, e_skey_id, e_skey_val, e_skey_sig |
| A05 (name, byte[]) | bytes → HSJ.A00 = RFC3986 percent-encoding, UPPERCASE HEX | id, backup_token |
| A00 (name, int) | 0→"false", 1→"true" | clicked_education_link and others |
| A06(map) | values (byte[]) → HSJ.A00 percent-encoding | the entire additional verify-flow map (entered in A0I…A0S) |
| GCO.A1F / AbstractC87053t4.A1b | string → UTF-8 bytes (for the map) | during map assembly |

## Value selection conditions

- `rc="0"` — always (RELEASE, X/C1Io; A0Z:649–657).
- `method` is absent from /v2/register in a normal registration: r31 (verify$2)
  receives a value only for alternative flows (send_sms, silent_auth,
  recaptcha, oauth_email, passkey, discoverable_credential, acc_tr,
  silent_auth_ts_43) or when codeVerificationMode==6 (dynamic 2FA); a fresh
  registration runs with mode=0 → null → A02 does not put the field (see
  README_code).
- `tos_version` — the regional AB experiment 19561 (ID/MX/BR/CO); see the
  chapter.
- The 2 hidden A0P parameters — only when flag 7999 is active.
- The Kotlin-bridge/wamsys switch — A0a (flags 24762/24763): with the flags
  off, the request goes through legacy-wamsys (C40916ICd), with the same
  parameter set.

## Cryptography and encoding

- Transport — wamsys/TLS with a domain-fronting list (A0J,
  DomainFrontingManager).
- Map values — UTF-8 → RFC3986 percent-encoding (HSJ.A00, uppercase HEX).
- The e2e bundle (authkey + e_*) — RawURL-Base64; id/backup_token —
  percent-HEX.
- Native integrity blobs (_gs/_gg/gpia/_gi/_ga/_gp/_ge) are encrypted with
  AES-256-CBC using the key SHA256(standard Base64(authkey_raw)) — see chapter
  05-integrity.

## Examples

A minimal skeleton of a correct /v2/register (values from the live dump):
`cc=7, in=9206309125, code=<OTP>, lg=ru, lc=RU, rc=0, entered=1,
mistyped=7, entrypoint=suma, has_play_store=true, passkey_login_status=
not_attempted, recaptcha={"stage":"ABPROP_DISABLED"}, mcc/mnc/sim_mcc/sim_mnc=000,
sim_type=0, simnum=0, hasinrc=1, network_radio_type=1, airplane_mode_type=0,
roaming_type=0, cellular_strength=5, device_ram=3,56, pid=<real PID>` +
the e2e bundle + id/backup_token + native blobs. No method, no tos_version,
no vname, no context (consumer flow, mode=0).
