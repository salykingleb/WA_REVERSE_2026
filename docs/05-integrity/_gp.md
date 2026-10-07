# _gp — hash of the manifest permission set (SHA256 of the sorted concatenation of 85 permissions)

## Place in the flow

The `_gp` parameter is a base64 string of the SHA-256 over the list of the application's requested permissions. It is attached
to all registration HTTP requests (`/v2/code`, `/v2/register`, makeConsentRequest, pre-chatd
AB check, verifySecurityCode, registerPhoneNumber, LinkedUsersActivity) by the native package:

```
X/IF6.java:553-567 (A0S):
    JniBridge.jvidispatchIOO(7, app, waj)                 // initialization
    map2 = JniBridge.jvidispatchOOO(16, app, waj)         // the underscore-parameter map (_gp is in it)
    map.putAll(map2)
```

There is no `_gp` string in the Java/DEX layer (a scan of the string tables of all classes*.dex; control: `gpia` and `authkey`
are found by the same scan). The value is formed natively; the list source is PackageManager via JNI
(the merged superpack library's string table contains `getPackageManager` and `getPackageInfo` with the
signature `(Ljava/lang/String;I)Landroid/content/pm/PackageInfo;` — stream_000 @0x38816d, @0x393295).
After login the same value (byte-for-byte) is embedded into the XMPP integrity_payload (the `_gp` field).

## Wire format (+ live example)

```
_gp = percent-escape( base64.Std( SHA256( concat( sort(requestedPermissions) ) ) ) )
```

Live value (SM-A325F, 2.26.35.75, Android 12+; IDENTICAL in /v2/code and /v2/register):

```
XpccAx8EKQswlTL2MtpCnGIMXTCaUbAf95ElLECc6jk=
```

On the wire it is percent-encoded: `XpccAx8EKQswlTL2MtpCnGIMXTCaUbAf95ElLECc6jk%3D` (`=`→`%3D`;
if the base64 contained `+`/`/` — `%2B`/`%2F` UPPERCASE).

## How it is formed in the application

The exact formula (PROVEN by a byte-for-byte match with the live value, a brute-force of 105 994 variants):

1. **The list**: all requested permissions returned by
   `PackageManager.getPackageInfo("com.whatsapp", GET_PERMISSIONS=0x1000)` on Android 12+ —
   `PackageInfo.requestedPermissions` / `getRequestedPermissions()`. These are the 86
   `uses-permission` entries of the binary AndroidManifest.xml of base.apk (83 regular + 3
   `uses-permission-sdk-23`: CALL_PHONE, ANSWER_PHONE_CALLS, READ_CALL_LOG) **MINUS**
   `android.permission.BLUETOOTH` — the legacy permission is silently dropped by the platform's normalization
   when BLUETOOTH_CONNECT is installed. Total **85 strings** (verified by a diff with the live
   `dumpsys package` and the app_process dumper PermDump on the device: the system returns exactly 85,
   the order is exactly the manifest order).
2. **Sorting**: lexicographic by name (Java `String.compareTo`/TreeSet; for ASCII names —
   byte-wise, equivalent to `sorted()` in Python).
3. **Concatenation**: concatenation of the sorted strings **WITHOUT a separator** (join ""), UTF-8, no NUL,
   no lengths.
4. **Hash**: plain SHA-256 (32 bytes).
5. **Encoding**: base64.Std (with `=`) → percent-escape (X/HSJ.A00 / url.QueryEscape — equivalent
   on this charset).

Hypotheses checked and REJECTED by brute force (gp_bruteforce.py): manifest order without sorting
(yields `kuLlzOrafTqa6LGNAb62lrNpFR+DperZgTNgxGuvSVM=` ≠ the target); the granted subset (52 granted —
the grant state does NOT participate, _gp depends only on the full requested list); separators
(`,`/`;`/`\n`/`|`/space/`:` and others); lower-case/short names/without the `android.permission.` prefix;
UTF-16LE; double SHA-256; SHA-256 of the hexdigest; concatenation of per-element digests;
HMAC-SHA256 with the authkey/integrity AES key as keys. The only match with the live value is
`requested85 | sorted | plain | sep='' | join | utf8 | sha256`.

The reference 85 (2.26.35.75, consumer APK; the full file is apk/manifest_permissions_full.txt):

```
RECEIVE_SMS, ACCESS_NETWORK_STATE, AUTHENTICATE_ACCOUNTS, GET_ACCOUNTS,
FOREGROUND_SERVICE_DATA_SYNC, VIBRATE, RECEIVE_BOOT_COMPLETED, ACCESS_COARSE_LOCATION,
ACCESS_FINE_LOCATION, CAMERA, RECORD_AUDIO, READ_EXTERNAL_STORAGE, DETECT_SCREEN_CAPTURE,
INTERNET, com.google.android.apps.aicore.service.BIND_SERVICE, BLUETOOTH_CONNECT,
ACCESS_WIFI_STATE, CHANGE_WIFI_STATE, NEARBY_WIFI_DEVICES, READ_PHONE_STATE,
READ_PHONE_NUMBERS, FOREGROUND_SERVICE_MICROPHONE, FOREGROUND_SERVICE_CAMERA,
FOREGROUND_SERVICE_MEDIA_PROJECTION, FOREGROUND_SERVICE_PHONE_CALL, SCHEDULE_EXACT_ALARM,
USE_BIOMETRIC, USE_FINGERPRINT, com.android.vending.BILLING, WRITE_EXTERNAL_STORAGE,
MANAGE_OWN_CALLS, CALL_PHONE, MODIFY_AUDIO_SETTINGS,
com.google.android.finsky.permission.BIND_GET_INSTALL_REFERRER_SERVICE, DETECT_SCREEN_RECORDING,
FOREGROUND_SERVICE_MEDIA_PLAYBACK, ACCESS_MEDIA_LOCATION, BROADCAST_STICKY, CHANGE_NETWORK_STATE,
FOREGROUND_SERVICE_LOCATION, GET_TASKS, INSTALL_SHORTCUT, MANAGE_ACCOUNTS, NFC, READ_CONTACTS,
READ_BASIC_PHONE_STATE, READ_PROFILE, READ_SYNC_SETTINGS, READ_SYNC_STATS, SEND_SMS,
USE_CREDENTIALS, WAKE_LOCK, WRITE_CONTACTS, READ_MEDIA_AUDIO, READ_MEDIA_IMAGES,
READ_MEDIA_VIDEO, READ_MEDIA_VISUAL_USER_SELECTED, POST_NOTIFICATIONS, WRITE_SYNC_SETTINGS,
REQUEST_INSTALL_PACKAGES, FOREGROUND_SERVICE, USE_FULL_SCREEN_INTENT,
com.android.launcher.permission.INSTALL_SHORTCUT, com.android.launcher.permission.UNINSTALL_SHORTCUT,
com.google.android.c2dm.permission.RECEIVE, com.google.android.gms.permission.AD_ID,
com.google.android.providers.gsf.permission.READ_GSERVICES,
com.sec.android.provider.badge.permission.READ, com.sec.android.provider.badge.permission.WRITE,
com.htc.launcher.permission.READ_SETTINGS, com.htc.launcher.permission.UPDATE_SHORTCUT,
com.sonyericsson.home.permission.BROADCAST_BADGE,
com.sonymobile.home.permission.PROVIDER_INSERT_BADGE,
com.huawei.android.launcher.permission.READ_SETTINGS,
com.huawei.android.launcher.permission.WRITE_SETTINGS,
com.huawei.android.launcher.permission.CHANGE_BADGE, com.whatsapp.permission.BROADCAST,
com.whatsapp.permission.MAPS_RECEIVE, com.whatsapp.permission.REGISTRATION, com.whatsapp.sticker.READ,
ANSWER_PHONE_CALLS, READ_CALL_LOG, RUN_USER_INITIATED_JOBS, com.facebook.services.identity.FEO2,
com.whatsapp.DYNAMIC_RECEIVER_NOT_EXPORTED_PERMISSION
```

(all with the `android.permission.` prefix, except the explicit com.*; sorting happens before hashing,
so the list's storage order does not matter).

## Native implementation (libwhatsapp.so / merged lib superpack)

- Issuance — the map assembler @0x7cfdd3ffd4 (the dispatcher `jvidispatchOOO(16)`), as with _ga/_ge.
- The PackageManager JNI infrastructure in the superpack merged library (libs.so, unpacked into 140
  xz streams, stream_000): the strings `getPackageManager`, `getPackageInfo` with the signature
   `(Ljava/lang/String;I)Landroid/content/pm/PackageInfo;` — the native code queries PackageInfo
   with the GET_PERMISSIONS flag (the list source is the requested permissions in manifest order).
- The `_gp` parameter name is NOT open in the merged library's table — in its place is a short binary
  blob (the 21-byte `2e1e26113f472a531a4f0c1543082204324b576f6b` @stream_000+0x5b41d4, between
  `event_name` and `sent`): sensitive names are encrypted by a separate scheme (XOR/Caesar/keys ≤3 B
  were brute-forced — not it; cracking it requires an IDA analysis of the merged library's string decryptor).
- SHA-256 and base64 are native libwhatsapp chains (init 0x7cfdd71df4 / update 0x7cfdd71e44 /
  final 0x7cfdd71e94 → base64(out,32)), or their analogue in the merged library.

## Value selection conditions

- The list depends on the APK version (the manifest) and on the platform: Android 12+ → 85 (without BLUETOOTH);
  Android < 12 → 86 (with BLUETOOTH) and, accordingly, a DIFFERENT hash.
- It does NOT depend: on the permissions' grant state (installed/granted — irrelevant), on the device,
  on the authkey, on the moment of the request.
- **Stability**: _gp is identical in /v2/code and /v2/register of one installation; in the 2.26.37.73 login
  integrity_payload `_gp` is byte-for-byte equal to the registration value of THE SAME
  installation (the 35.75 profile → `XpccAx8E…`); when the application version changes, the list is taken from the new
  manifest and the hash changes.
- Since the formula is deterministic (sort+join+sha256), accidental shuffling of the list is ruled out:
  the live hash is stable and reproducible by sorting.

## Cryptography and encoding

- SHA-256 of the sorted concatenation of the 85 permission names without a separator (UTF-8).
- base64.StdEncoding (alphabet `A-Za-z0-9+/`, padding `=`).
- Percent-encoding uppercase: `+`→`%2B`, `/`→`%2F`, `=`→`%3D` (X/HSJ.A00 on the phone ==
  url.QueryEscape on this charset).
- Inside integrity_payload the value goes as a plain base64 string `"XpccAx8E…"` (without percent).

## Examples

- The target live value: `XpccAx8EKQswlTL2MtpCnGIMXTCaUbAf95ElLECc6jk=` = SHA256(''.join(sorted(85))).
- The control counter-example (manifest order without sorting): `kuLlzOrafTqa6LGNAb62lrNpFR+DperZgTNgxGuvSVM=` — NOT equal to the live one, proving that sorting is mandatory.
- On a version update: the list = all uses-permission of the new manifest minus legacy BLUETOOTH
  (when BLUETOOTH_CONNECT is present on Android 12+), sorting, concatenation, SHA-256, base64.
