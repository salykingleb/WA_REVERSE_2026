# _ga — installation ages block and anti-monkey flags (open JSON {bi, ap, ai, mp, ae, mu})

## Place in the flow

The `_ga` parameter (native name sid 80 from the obfuscated table) is attached to all registration
HTTP requests (`/v2/code`, `/v2/register` etc.) by the native package of dispatcher 16:

```
X/IF6.java:553-567 (A0S):
    JniBridge.jvidispatchIOO(7, app, waj)                 // initialization
    map2 = JniBridge.jvidispatchOOO(16, app, waj)         // the underscore map, _ga is in it
    map.putAll(map2)
```

The _ga builder is called from the underscore-parameter map assembler **@0x7cfdd3ffd4** (the dispatcher
jvidispatchOOO(16)); immediately after the call — put(map, sid 80 `_ga`) @0x7cfdd40298; a separate gate —
AB flag 16 at 0x7cfdd2c408. In the Java/DEX layer there are no `_ga` strings (a scan of the string tables of all dex); the values
arrive as ready byte[]-ASCII. After login the same object is embedded into the XMPP integrity_payload
(the `_ga` field in the payload key order).

**Important**: on the wire _ga is an OPEN JSON (not a blob); only the nested bi is encrypted.

## Wire format (+ live example)

The wire key order: `bi, ap, ai, mp, ae, mu` (the native builder inserts in the order
mu→mp→ae→ai→ap→bi — @0x7cfdd30864; the serializer reorders to the wire order). Numbers are integers,
booleans are the literals false/true, the JSON is compact.

Live examples (both sessions):

```json
{"bi":"Bge8ej5JWBp5Fi0v9KnANajtzs0+L45Q8FyEswRJUhzNsgKZyyweGl5KFNk//MVHQSxziPIlHMAHtfI+ST1LBA==","ap":1024549,"ai":1649812,"mp":false,"ae":1649768,"mu":false}
{"bi":"kjqKmWIZCS3Popc6kjtSSqz4Hnf/l8HKpSQ8k2b/JysWw38spnGRCBxIAF+CNJ1v6xnoJTb4BZVRHastSyoMUw==","ap":351098,"ai":80553,"mp":false,"ae":7419,"mu":false}
```

(in the first, HTTP, variant the two `/` inside bi are escaped as `\/`; in the second — the XMPP integrity_payload,
where bi is inserted AFTER escaping — the slashes are bare.) Then the whole JSON is percent-encoded:
`{`→`%7B`, `"`→`%22`, `:`→`%3A`, `,`→`%2C`, `\/`→`%5C%2F`, `+`→`%2B`, `=`→`%3D`.

## How it is formed in the application

The semantics of all fields is settled at the assembly level + import symbols (D_ga.md) — ap/ai/ae are
**FILE AGES IN SECONDS** (wall-clock: `time() − st_mtim.tv_sec`), not uptime and not counters:

| Field | Source (proven) | Meaning |
|---|---|---|
| `bi` | `WAJIntegrityCreateEncryptedAES256CBC(x2)` @0x7cfdd3b3a4 from the builder's argument | the encrypted blob; plaintext = base64.Std(32 random bytes) — a 44-character string |
| `ap` | `time(NULL) − stat(ApplicationInfo.sourceDir).st_mtim.tv_sec` (0x7cfdd3a534; stat-PLT 0x7cfe3449f0) | the age of base.apk, seconds since the last install/update |
| `ai` | `time(NULL) − MIN(st_mtim)` over the ApplicationInfo.dataDir tree (0x7cfdd30f88 w2=sid 43 "dataDir"; the traversal routine 0x7cfdd3a2d0 depth≤10) | the age of the oldest element of the private data directory ≈ seconds since the install/last data wipe |
| `ae` | the same MIN(mtime) over `Context.getExternalFilesDir(null)` (0x7cfdd3113c) | the age of the oldest element of the external directory ≈ ai minus the first-launch lag; if the path is unavailable, the key may be ABSENT |
| `mp` | a scan of `/proc/<pid>/cmdline` (readdir /proc, atoi pid; sid 41 `/proc/%s/cmdline`) for the string `com.android.commands.monkey` (sid 39, MCFStringFind) @0x7cfdd30cc0 | "monkey process": a monkey is running on the device |
| `mu` | static `android/app/ActivityManager.isUserAMonkey()` (()Z) @0x7cfdd30bb4 | "monkey user": the application itself is running under monkey/uiautomator |

Numeric serialization — `MCFNumberCreate(1,&v)` (0x7cfe344870), insertion — the wrapper
`MCFDictionarySetValueForCStringKey` (0x7cfdd2fbbc); booleans — `MCFBooleanTrue/False`
(0x7cfe344e40/0x7cfe344e50). sid names: ai=94, ap=95, ae=96, mu=97, mp=98, bi=99, _ga=80.
Time is read via PLT 0x7cfe343b50 (GOT 0x7cfe379628 → bionic `time`), mtime from the stat buffer at
offset **+0x58** (arm64 st_mtim.tv_sec).

### bi (the inner blob)

- plaintext: base64.Std of 32 random bytes — exactly 44 characters, std alphabet (`+`, `/`), one `=`.
  Live plaintext: `nZSX05b23Fax2H/ZCTptouz3dcvpv/ifppGSfUfqXnM=` (hex
  `9d9497d396f6dc56b1d87fd9093a6da2ecf775cbe9bff89fa691927d47ea5e73`).
- envelope: AES-256-CBC with the integrity key (SHA256(std-base64(authkey))), IV 16 B random ‖ PKCS7
  (44→48, padding 4×0x04), base64.Std — the result is exactly 88 characters with `==` (64 bytes = IV16‖ct48).
- Live blob: `Bge8ej5JWBp5Fi0v9KnANajtzs0+L45Q8FyEswRJUhzNsgKZyyweGl5KFNk//MVHQSxziPIlHMAHtfI+ST1LBA==`.

## Native implementation (libwhatsapp.so, base 0x7cfd633000)

- The _ga builder **@0x7cfdd30864**: mu ← 0x7cfdd30bb4; mp ← 0x7cfdd30cc0; ae ← 0x7cfdd3113c;
  ai ← 0x7cfdd30f88(sid 43); ap ← 0x7cfdd30f88(sid 42 "sourceDir"); each path → 0x7cfdd30a18
  (lstat 0x7cfe344ec0; S_ISDIR(0x4000): directory → 0x7cfdd3a248 {min=time(); the traversal routine
  0x7cfdd3a2d0(path,&min,depth≤10): lstat + readdir (skipping `.`/`..`) + stat of each element,
  S_IFREG(0x8000) → comparison of mtime(+0x58) with min, recursion depth+1}; file → 0x7cfdd3a534
  {stat; time() − mtime}); bi ← 0x7cfdd3b3a4 (= `_WAJIntegrityCreateEncryptedAES256CBC`).
- The map assembler @0x7cfdd3ffd4, put `_ga` @0x7cfdd40298, gate AB 16 @0x7cfdd2c408.
- The field-name strings are the AES-128-ECB-obfuscated table 0x7cfdd76a64 (the sid 93-103 cluster:
  93=`/proc/sys/kernel/random/boot_id`, 94-99 = ai/ap/ae/mu/mp/bi, 100-103 = sv/sb/aid/did).

## Value selection conditions

Three live measurements and their interpretation (all quantities are seconds, a single "now" point):

| Measurement | ap | ai | ae | ai−ae | Comment |
|---|---|---|---|---|---|
| OLD (early) | 704785 (8.16 days) | 11651175 (134.85 days) | 11650667 | 508 s | an old install, an old apk update |
| NEW 2026-09-22 | 1024549 (11.86 days) | 1649812 (19.10 days) | 1649768 | 44 s | data wiped ~2026-09-02, the version updated ~2026-09-10 |
| 2.26.37.73 (registration) | 351098 (~4.06 days) | 80553 (~0.93 days) | 7419 | 73134 s (~20.3 h) | ap > ai — the "Clear data" case without a reinstall |

Invariants of a real device (server-verifiable):
- usually `0 < ap ≤ ai` (the apk is written no earlier than the dataDir is created); `ap > ai` is possible only after
  "Clear data" without a reinstall — a rare but live case (the 37.73 measurement);
- `ap ≤` the age of the release of the declared APK version (the version's file is no older than its
  release — verifiable against User-Agent/H);
- `ae < ai` is MANDATORY (the external directory is created on the first launch); ai−ae is usually small
  (tens-hundreds of seconds, live 44 and 508), but can reach ~20 h (live 73134);
- there is no fixed ai/ap ratio (live 16.5 and 1.6) — the anchors are independent;
- mp=false, mu=false always on a non-monkey device (all live captures); true — only under
  monkey/uiautomator.

Freshness: _ga (and bi) in /v2/code and /v2/register are **byte-for-byte identical** — the values are cached/reused
within a registration (a recomputation would fall on the same second anyway).

## Cryptography and encoding

- The whole _ga is an open ASCII JSON → percent-encoding (HSJ.A00: %XX UPPERCASE; `\/`→`%5C%2F`).
- Only bi is encrypted: AES-256-CBC, key = SHA256(standard base64(authkey)), IV 16 B random
  ‖ PKCS7, base64.Std (see encryption-key.md).
- The escaping `/`→`\/` in the HTTP variant also affects bi's slashes (live confirmation
  `KFNk\/\/MVHQ`); in integrity_payload bi is inserted after escaping — bi's slashes are bare.
- Byte-for-byte reproducibility of the chain (b64 → replace(`/`→`\/`) → JSON → percent-escape) on the live
  value is confirmed (== True).

## Examples

- "Used device" model: ai = logUniform(9e5…1.2e7) s; ap = ai − uniform(6e4…ai), but not older than
  the version's release; ae = ai − uniform(30…900) s; mp=mu=false; bi — a fresh envelope of 32 random bytes.
- "Fresh install" model (the typical case of a new registration): ai ∈ 120…7200 s; ap ≈ ai ± 300;
  ae = ai − 30…120.
- UNACCEPTABLE: ap > ai on a "clean" profile; ap older than the release of the declared version; ai−ae in the
  hundreds of thousands of seconds; statistically identical ap/ai/ae across hundreds of registrations (a cluster is a sign of a farm).
