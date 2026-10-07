# SafetyNet envelope: `<ib><integrity_payload>` (768 bytes)

WhatsApp 2.26.35.75 (263507522). Wire: live capture of the first login; the node is sent at 02:48:52.780 — 35 ms after the server's `<safetynet><integrity nonce='ATawnSKrOzJC7B6lySW4LiPRrfNIcHONk0ZDrKqfYItuJDzO7zxxdIT7P5lV-Mw6koYe7wvttubvbRklSiv8_-YH'>` (nonce 66 bytes = 88 b64url chars, .745). The body is a base64 blob of 1024 characters = **768 bytes = exactly 48 AES blocks**. This is NOT a JWS: the base64 has no dot, and after decoding there is no `eyJ` prefix. There is no plaintext 66-byte nonce on the wire.

## Native path: the JNI dispatcher, id 5

The entry point is the shared JNI method `Java_com_whatsapp_wamsys_JniBridge_jvidispatchIOOOO`:

- code: VA **0x7cfdd1dd00**; name slot in the `RegisterNatives` table (rodata2): **0x7cfe363a90**;
- JNI signature: `(ILjava/lang/Object;Ljava/lang/Object;Ljava/lang/Object;Ljava/lang/Object;)J`;
- the `w2` register after `JNIEnv*`/`jclass` is the operation number; the four objects are saved in `x23`, `x24`, `x21`, `x22`.

The id dispatch branch (fragment):

```text
0x7cfdd1dd2c  cmp  w2, #2
0x7cfdd1dd34  b.gt 0x7cfdd1dda4      ; 3, 4, 5
…
0x7cfdd1dda4  cmp  w2, #3
0x7cfdd1dda8  b.eq id3               ; 0x7cfdd1de70
0x7cfdd1ddac  cmp  w2, #4
0x7cfdd1ddb0  b.eq id4               ; 0x7cfdd1df24
0x7cfdd1ddb4  cmp  w2, #5
0x7cfdd1ddb8  b.ne done
```

id 5 → wrapper `0x7cfdd3ff30` → stub `0x7cfdc78280` → the body of **`_WCAAPIInitiatePeriodicIntegrityRequest`** at **0x7cfdd3ffd0** (name string in rodata: `0x7cfd7975d5`). The body is **308 instructions**, all calls local, **with no trip to Google** — hence the 35 ms on the wire.

Before that, Java calls the preparation — the other method `jvidispatchIIIIDOOO` (code `0x7cfdd21044`, callee `0x7cfdd31dd8`), handler 179 in `X/C1US` (DI case 126), posted from `X/C27871Nj` → `C1HI.A1I` (log `on-attestation-request`), then `DTA` case 10:

```java
int i = c22350zE.A01.A0w(12964) ? 3 : 0;
JniBridge.jvidispatchIIIIDOOO(
    i, 62949436L, 855397460L, 796.6509679599703d,
    context, jniBridge.getWajContext(), new byte[20]);
JniBridge.jvidispatchIOOOO(5, nonce, context, new CN3(…), jniBridge.getWajContext());
```

Flag **12964** selects the first argument of the preparation (3 or 0). The constants **62949436** (cloud project number) and **855397460** and the empty 20 bytes are not the session nonce. The same call stands in the registration flow, where the mode is selected by flag **12965** (19 or 0). `0x7cfdd31dd8` writes the cloud project number into an already created `IntegrityTokenRequest` (control strings nearby in rodata: `SetCloudProjectNumber called with a null IntegrityTokenRequest`, `SetNonce called with a null IntegrityTokenRequest`, …).

## What the body 0x7cfdd3ffd0 assembles

The same helpers that assemble the registration `_gi`:

| Call from the body | Role |
|---|---|
| `0x7cfdd3b338` | `.so` directory walker (`.` / `..` / `.so`, errors `failed_sha`, `invalid_library_name`) |
| `0x7cfdd23ee0` | `WAJIntegrityDataStoreCopyValue` |
| `0x7cfdd31bfc` | AB config read |
| `0x7cfdd3a5c8` | DataStore value of type `0x2c` (nativeLibraryDir → `_lh`) |
| `0x7cfdd3aa18` | DataStore value of type `0x2b` (the same step as the `.so` walker's) |
| `0x7cfdd3c5e0` | DataStore value of type 0 |
| `0x7cfdd3b3a4` | `_WAJIntegrityCreateEncryptedAES256CBC` — the final encryption |

## Encryption

The envelope is AES-256-CBC (`_WAJIntegrityCreateEncryptedAES256CBC`, address `0x7cfdd3b3a4`, name string at `0x7cfd786c1a`), IV || ciphertext format, PKCS7 padding. The envelope key = `SHA256(std_base64(authkey))` — the same one used to encrypt the registration `gpia`/`_gi`/`_gg`: the formula is confirmed on two independent sessions (2.26.35.75 and 2.26.37.73); five blobs of one installation decrypt with a single key.

## Plaintext: the exact key order

The decrypted live blob of 2.26.37.73 (768 bytes, same as here — the structure is identical):

```json
{"aid":"VM60VU78\/F6NzE1CSRYipwztp4PCM\/2hlFoIdUZBb4g=",
 "_dh":"jjyxvURV8j4F8rILRZttTITKv0JCvJG6F52K32iZ\/Ag=",
 "_lh":"oGLRVumypzq8fegSIB+MKwj1iNrmarhbZYDdj8yLCns=",
 "_iln":"i7ls4vAnG2Bvmsp8NDKh7uREGcY4q8Hi4JtdCLix+Cc=",
 "_ln":"KMr1FDZ5Qv9UsYvUwaPmFmshuABXLq3rfxeELvAebKk=",
 "_ge":{"sb":false,"sv":false},
 "_gp":"XpccAx8EKQswlTL2MtpCnGIMXTCaUbAf95ElLECc6jk=",
 "_ga":{"bi":"kjqKmWIZ…+CNJ1v6xnoJTb4BZVRHastSyoMUw==","ap":351098,"ai":80553,"mp":false,"ae":7419,"mu":false},
 "_ip":"com.whatsapp",
 "_isb":"129472282",
 "_n":"ATbE74Q57yF6lIkJ5w6BK8vaZoMQS1TZjH_NbI4AciTkxFV2NMfCTPR0B-X6oz8yO4RbMoACgMnDl1p30Zr1lRzX",
 "_icr":"OKD31QX+GP7GT780Psqq8xDb15k=",
 "_is":"1oUpYgdN0ehe\/QGjX23NFK+oj8riGuiJQJIdhGCsJus="}
```

The key order is fixed: `aid, _dh, _lh, _iln, _ln, _ge, _gp, _ga, _ip, _isb, _n, _icr, _is`. Semantics of the fields:

| Field | Content |
|---|---|
| `aid` | installation identifier (stable, equal to the registration `aid`) |
| `_dh` | SHA-256(classes.dex) — the dex digest |
| `_lh` | SHA-256(the `.so` file from nativeLibraryDir); source — DataStore 0x2c |
| `_iln` | SHA-256(concat(sorted(libs.spo names))) — stable across versions |
| `_ln` | hash of the native library list (stable across versions) |
| `_ge` | environment flags `{"sb":…,"sv":…}` |
| `_gp` | 32B fingerprint (the installation's permissions profile) |
| `_ga` | installation counters: `bi` (a token from 32 random bytes encrypted with the same envelope), `ap`≤`ai`, `ae`≈ai−30..900, `mp`, `mu` |
| `_ip` | package name `com.whatsapp` |
| `_isb` | APK size in bytes |
| `_n` | **the server's safetynet nonce in plaintext** — the blob is bound to the request |
| `_icr` | SHA-1 of the APK signature in b64 (`OKD31QX+GP7GT780Psqq8xDb15k=`) |
| `_is` | SHA-256 of the APK |

JSON escaping: all `/` → `\/`, EXCEPT the slashes inside `_ga.bi` — `bi` is inserted after escaping.

## Sending and metrics

The result is returned to Java by a callback, `JniBridge.jnidispatchIOO` case 3:

```java
boolean zA0U = c0oe.A0U(
    new C0P3(new C0P3("integrity_payload", str3, null), "ib", null),
    194);
c07570Ws.A02 = "safety-net-attestation";
c07570Ws.A01 = zA0U ? "success" : "failed";
```

There is no `integrity_payload` literal in the `libwhatsapp` rodata — the string is supplied by Java in case 3. `CN3` memorizes `elapsedRealtime`; the `safety-net-attestation` metric (`success`/`failed`) measures the duration. On the wire the node went out as `<ib><integrity_payload>` at .780, 11 ms after `keystore_attestation` (.769).

## Reference sizes

| Session | Blob on the wire | AES blocks |
|---|---|---|
| 2.26.35.75, 2026-09-02 (.780) | 1024 b64 chars = 768 bytes | 48 |
| 2.26.37.73, 2026-09-28 (`вход после регистрации.txt`) | 768 bytes | 48 |

The identical size with a different set of fields means the JSON structure (including `_lh` and `_n`) is stable across versions.

## Where to look

- Dispatcher: `0x7cfdd1dd00`, slot `0x7cfe363a90`.
- id 5: wrapper `0x7cfdd3ff30`, stub `0x7cfdc78280`, body `0x7cfdd3ffd0`.
- Preparation: `jvidispatchIIIIDOOO` `0x7cfdd21044` → `0x7cfdd31dd8`.
- AES-CBC: `0x7cfdd3b3a4`, name at `0x7cfd786c1a`.
- Java: `X/C27871Nj.java` (~270–320), `X/C1HI.A1I`, `X/C1US` (179, DI-126), `X/DTA.java` case 10, `X/C22350zE.java`, `com/whatsapp/wamsys/JniBridge.java` (the `jnidispatchIOO` case 3).
- AB flags: 12964 (preparation before id 5), 12965 (registration mode).
