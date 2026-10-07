# The `_gs` parameter — environment security signals (WASafe)

**Application:** WhatsApp Android 2.26.35.75 (versionCode 263507522), arm64-v8a.
**Generator:** the native library `libwasafe.so` (its own logic) + the crypto dependency `libwasafedeps.so` (X25519, AES-GCM). Neither library ships inside the APK — both are fetched by the super pack and loaded on demand via `dlopen` (details in `libwasafe.md`).
**Format reverse-engineered on 2026-09-22**: memory dumps of a live process (Samsung SM-A325F, pid 8643), a complete disassembly pass over both libraries, 5 runtime runs of the native calls with interception of the plaintext/key/envelope.

---

## 1. Summary

| Property | Value |
|---|---|
| Purpose | anti-fraud scoring: "is this a live clean stock phone, or root/emulator/fake/tampering" |
| Wire format | JSON `{"em":"<base64url>"}`, ~223 characters, URL-encoded inside the body |
| Binary structure | 167 B = ephPub(32) ‖ AES-256-GCM ct(119) ‖ tag(16) |
| Plaintext | CSV of 19 fields, 119 B in live captures |
| Cryptography | X25519-ECDH with the server key → AES-256-GCM (nonce of 12 zeros, no AAD) |
| Freshness | ALWAYS new on every request: a new ephemeral pair, new timestamps, a new rand |
| Participation | `/v2/code`, `/v2/register` (and all registration requests: exist/consent/reset/passkey) |
| Storage | not persisted in client state, not reused |

---

## 2. Place in the flow

### 2.1. Which requests carry `_gs`

`_gs` is inserted into ALL registration requests via the shared collector `A0S` (see 2.2):

| Request | App method | Position of `_gs` on the wire |
|---|---|---|
| `/v2/code` | verifySecurityCode | 3rd: `method,backup_token,_gs,sim_mnc,id,...` |
| `/v2/register` | verifySecurityCode (entered=1) | 4th: `code,backup_token,sim_mnc,_gs,id,...` |
| `/v2/exist` | checkIfExists | present |
| `/v2/consent` | makeConsentRequest | present |
| resetSecurityCode, passkeyAuthResult | — | present |

### 2.2. The Java bridge (the only path by which the parameter enters the request)

The Java code contains no `"_gs"` literal at all (grep over the decompiled sources — 0 references). The parameter comes from native:

```
X/IF6.java:553-567  A0S(IF6, Map):
  1) JniBridge.jvidispatchIOO(7, application, wajContext)   // warm-up: lazy load of libwasafe.so + signal collection
  2) Map map2 = (Map) JniBridge.jvidispatchOOO(16, application, wajContext)
  3) map.putAll(map2)                                        // 7 keys go straight into the request parameters
```

The native map of selector 16 contains exactly 7 keys (string IDs of the obfuscated libwhatsapp.so table):
`"_gg"(84), "_gi"(87), "_gp"(88), "_ga"(89), "_ge"(90), "aid"(102), "_gs"(91)`. The `_gs` value is an already finished `{"em":...}` string, assembled and serialized on the native side.

`A0S` call sites in `X/IF6.java`: lines 694 (makeConsentRequest), 804 (resetSecurityCode), 1427 (checkIfExists), 1576 (verifySecurityCode — /v2/code and /v2/register), 1698 (passkeyAuthResult).

### 2.3. Freshness (live capture 2026-09-22)

In a code→register request pair from the same phone, all long-lived parameters (id, backup_token, aid, _gp, _ga, authkey, fdid, expid, ...) are identical, while `_gs` (together with `_gg`, `gpia`, `_gi`, `client_metrics`) is **recreated**: a different plaintext, a different ephemeral key, a different envelope — but the SAME length (223 chars / 167 B / plaintext 119 B in both).

---

## 3. Wire format

```
_gs={"em":"FiAYvk_VQTXVsqVCkH9m5qcYnUI-7r14988SFxlcT2pNwylHs3miqD7..."}
```

- JSON with a single field `em` (the key `"em"` is string ID 92 of the libwhatsapp.so table);
- `em` = base64url (alphabet `A–Z a–z 0–9 - _`, RawURLEncoding, without `=`);
- decodes into 167 B: `ephemeral_X25519_pub(32B) ‖ AES-256-GCM_ct(119B) ‖ tag(16B)` (in the capture brief ct+tag appear as "135 B ct");
- 167 B = 55 full triplets × 4 chars + 2 remaining × 3 chars = **223 characters** without padding;
- the whole value is URL-encoded inside the request body: `%7B%22em%22%3A%22…`;
- envelope length limit on the libwhatsapp.so side: `< 0x800100`.

## 4. Plaintext (before encryption)

A CSV string of **19 fields**. Assembly is a chain of `__vsnprintf_chk` with the formats `"%d"`, `",%d"`×13, `",%u"`, `",%d"`, `",%d,%d"`, `",%d"`:

```
1,21469,21445,20541,21512,21510,21532,21526,21552,21390,21567,21543,21551,21719,20527,1513935229,198617515,20793,21926
```

| # | Field | Formula / meaning | Format |
|---|---|---|---|
| 1 | ver | the literal `1` — format version prefix | %d |
| 2–14 | stat codes ×13 | `signed_int32( code_i' ^ (base + i*7) )`, where `code_i' = code_i` when `stat()` succeeds, otherwise `code_i + 1000`; base codes `{10, 11, 0, 100, 101, 102, 103, 104, 1, 105, 106, 107, 108}` (index i = 0..12) | %d |
| 15 | base | obfuscation salt: `((unix_sec ^ usec) & 0xefff) + 0x1000`; range **[0x1000, 0xFFFF] = [4096, 65535]** (the 0xefff mask clears only bit 12); the only unsigned field | %u |
| 16 | rand | a 31-bit `libc rand()` after `srand(sec2 ^ usec2)` (second gettimeofday), XOR `(base + 91)` | %d |
| 17 | millis_lo | the low 32 bits of `tv_sec*1000 + tv_usec/1000` (UnixMilli, second gettimeofday), XOR `(base + 98)` | %d |
| 18 | millis_hi | the high 32 bits of the same time, XOR `(base + 105)` | %d |
| 19 | prepared | `*(u32*)0xebd8` in libwasafe.so .data: **1337** (0x539) by default, **4242** (0x1092) after `WASafeSignalsPrepare()`, XOR `(base + 112)` | %d |

Important details:
- all XOR fields are printed as **signed int32** (they can be negative; in practice only millis_lo is ever negative — when bit 31 of uint32(ms) is 1, which happens during half of every 49.7-day cycle of 2^32 ms);
- `usec` = `gettimeofday().tv_usec`; the FIRST gettimeofday call feeds base, the SECOND feeds millis/rand (the numbers end up adjacent, but not identical);
- the plaintext buffer is `malloc(0xf0)` = 240 B; the plaintext limit in the crypto dependency is 0x2800.

### 4.1. Table of the 13 stat paths (complete)

The order of the paths is confirmed byte-by-byte by deobfuscation of the libwasafe.so string table (a string's salt = the index of the stat check):

| i | Base code | Path | What a successful stat detects |
|---|---|---|---|
| 0 | 10 | `/sys/module/vmw_pvscsi` | VMware (guest parallel SCSI driver) |
| 1 | 11 | `/sys/module/vboxsf` | VirtualBox (shared folders) |
| 2 | 0 | `/system/build.prop` | a legitimate marker — present on any real Android (the "liveness" reference) |
| 3 | 100 | `/system/app/Superuser.apk` | root (superuser manager) |
| 4 | 101 | `/sbin/su` | root |
| 5 | 102 | `/system/bin/su` | root |
| 6 | 103 | `/system/xbin/su` | root |
| 7 | 104 | `/data/local/xbin/su` | root (manually placed) |
| 8 | 1 | `/proc/version` | kernel (stat-accessible in a privileged/emulated environment) |
| 9 | 105 | `/data/local/bin/su` | root |
| 10 | 106 | `/system/sd/xbin/su` | root |
| 11 | 107 | `/system/bin/failsafe/su` | root (engineering/failsafe build) |
| 12 | 108 | `/data/local/su` | root |

Semantics: `stat(path) == 0` → success → `code` goes into the field; `stat != 0` or path==NULL (the string decoder is poisoned) → **`code + 1000`**.

### 4.2. Inversion: what a clean phone sees

On a clean stock phone the su paths and VM modules **do not exist** → stat fail → `code+1000`. A success on a su path = root, on vmw/vbox = emulator, on /proc/version = a suspicious environment. The only mandatory success is index 2 (`/system/build.prop`), the "beacon of a live Android".

**Reference profile (a real Samsung SM-A325F, Android, root hidden by Magisk):** only index 2 succeeds (code 0), the other 12 fail (codes 1010, 1011, 1100, 1101, 1102, 1103, 1104, 1001, 1105, 1106, 1107, 1108 — base+1000). Some nuances of the reference: `/proc/version` is not accessible to `stat()` from the untrusted_app context (SELinux), and the su paths are hidden by Magisk — that is, the stat picture reflects what the app process itself sees, not what is physically present on the device.

## 5. Native implementation (addresses)

The generation of the `_gs` value is entirely native. Address summary (VAs of the dumped process for libwhatsapp.so; offsets from the module base for libwasafe.so / libwasafedeps.so):

| What | Address | Comment |
|---|---|---|
| `jvidispatchIOO` (native) | 0x7cfdd1d9fc | warm-up, selector 7 |
| `jvidispatchOOO` (native) | 0x7cfdd1e930 | jump table of 18 selectors @ 0x7cfd92622d |
| selector 7 → AB gates 15/16 | 0x7cfdd1dc44 → 0x7cfdd2c390 → 0x7cfdd2c404 | gate values 22925/24048 |
| WASafe signal collection | 0x7cfdd2f3e8 | Create→Get→insert→Release |
| `EnsureWasafeLoaded` (dlopen+dlsym×4) | 0x7cfdd2f630 | the string `"libwasafe.so"` @ 0x7cfd702c32 |
| export slots (.bss) | 0x7cfe391240 | +0 Create / +8 GetBase64 / +0x10 Release / +0x18 Prepare |
| Create/Get/Release/Prepare wrappers | 0x7cfdd2f82c / 0x7cfdd2f87c / 0x7cfdd2f8d0 / 0x7cfdd2f920 | the Prepare wrapper runs under a QPL timer |
| selector 16 map assembly | 0x7cfdd2c59c | getters 0x7cfdd2c628…0x7cfdd2c6b8 |
| insertion of `{"em":...}` into the map | 0x7cfdd2fbbc | key `"em"` = string ID 92 |
| JSON serialization | 0x7cfdd2fc10 / 0x7cfdd42d6c | the same serializer as client_metrics |
| plaintext assembly (libwasafe+0x50a0) | 0x50a0 | string decoding → 13× stat → 2× gettimeofday → vsnprintf |
| stat helper (libwasafe+0x56e4) | 0x56e4 | `stat(path)==0 ? code : code+1000` |
| envelope encryption (the library) | 0x64b8 | a wrapper over WASafeDepsCryptoEncryptBase64 |
| server X25519 key (copy in the library) | libwasafe rodata @0xd50 | 32 B as a flat blob |

The `"_gs"` key is itself a string of the obfuscated libwhatsapp.so table (AES-128-ECB over the "method name", decoder at 0x7cfdd76a64; length/offset tables at 0x7cfd9282b8/0x7cfd9287b0/0x7cfd928608/0x7cfd928b00).

## 6. Cryptography (the `em` envelope)

The server public key (the same one that encrypts the ENC of the registration parameters; in libwhatsapp.so it sits in rodata @0x7cfd9265e0, a copy inside libwasafe.so @0xd50):

```
8e8c0f74c3ebc5d7a6865c6c3c843856b06121cce8ea774d22fb6f122512302d
```

The `WASafeDepsCryptoEncryptBase64` pipeline (confirmed by runtime interception of mbedtls_gcm_setkey and the internal X25519):

1. `mbedtls_ctr_drbg_random(32)` → the ephemeral private key (DRBG personalized with the string `"wa_crypto"`); a NEW one for every `_gs`;
2. X25519 clamping: `priv[0] &= 0xf8; priv[31] &= 0x3f; priv[31] |= 0x40;`
3. `ephPub = X25519(priv, base point)` (the X25519 implementation is static inside libwasafedeps.so, function +0x597c, base point `09 00…00` @0x100a);
4. `shared = X25519(priv, 8e8c…302d)` — **raw secret, NO KDF** (HKDF/extract are absent) → used directly as the 256-bit key;
5. `mbedtls_gcm_setkey(AES, shared, 256)`; `mbedtls_gcm_crypt_and_tag(ENCRYPT, len, IV = 12 ZERO bytes, AAD = NULL/0, tag = 16B)` — one-time use is ensured by the ephemeral pair, not by the nonce;
6. envelope = `ephPub(32) ‖ ct ‖ tag(16)` → `MCIBase64EncodeURLSafeCreateASCIIString` (base64url without padding);
7. `platform_zeroize` wipes priv and shared immediately after use.

Runtime proof (3 runs on the device): envelope 223 chars → 167 B = 32+119+16; AES-256-GCM with the intercepted key and IV=0x00×12 without AAD decrypts the envelope into exactly the intercepted plaintext; X25519(intercepted priv, 8e8c…) == the native GCM key; the first 32 B of the envelope == pub(priv). The scheme is proven byte-by-byte.

## 7. Examples

### 7.1. Worked example (step by step)

Inputs: `unix = 1791200000` (0x6AC38B00), `usec = 314159` (0x4CB2F), `rand = 0x5A3C81F7`, the reference profile (only stat[2] succeeds), `prepared = 1337`.

```
unix ^ usec            = 0x6AC38B00 ^ 0x0004CB2F = 0x6AC7402F
0x6AC7402F & 0xefff    = 0x402F                  (bit 12 cleared)
base                    = 0x402F + 0x1000 = 0x502F = 20527
masks: base+91=20618, base+98=20625, base+105=20632, base+112=20639
codes': [1010,1011,0,1100,1101,1102,1103,1104,1001,1105,1106,1107,1108]  (all +1000, except i=2)

field 2 = 1010 ^ (20527+0)  = 1010 ^ 20527  = 21469
field 3 = 1011 ^ (20527+7)  = 1011 ^ 20534  = 21445
field 4 =    0 ^ (20527+14) = 0    ^ 20541  = 20541
... (i=3..12 analogous, mask step +7)
field 14 = 1108 ^ (20527+84) = 1108 ^ 20611  = 21719
field 15 = base (unsigned)                            = 20527
field 16 = 0x5A3C81F7 ^ 20618 = 1513948151 ^ 0x508A  = 1513935229
field 17 = uint32(1791200000314) ^ 20625             = 198617515
field 18 = (1791200000314 >> 32) ^ 20632 = 417^20632 = 20793
field 19 = 1337 ^ 20639                              = 21926
```

Result (118 B; millis_lo is 9 digits here):

```
1,21469,21445,20541,21512,21510,21532,21526,21552,21390,21567,21543,21551,21719,20527,1513935229,198617515,20793,21926
```

### 7.2. Live capture from the device (SM-A325F, 2026-09-22 04:27:07.522 UTC)

The plaintext captured by a hook on `WASafeDepsCryptoInputSetData` (119 B, 19 fields):

```
1,63950,63920,64074,65053,65045,65041,65065,65085,63901,65066,65232,65242,65220,64060,1566506798,-950080228,64261,65429
```

Reverse inversion (verified numerically):
- base = 64060 (0xFA3C);
- codes = `[1010,1011,0,1100,1101,1102,1103,1104,1001,1105,1106,1107,1108]` — only index 2 succeeds (`/system/build.prop`), all the rest are +1000;
- millis = (416 « 32) | 3344832386 = 1790051227522 → **2026-09-22 04:27:07.522 UTC** — matches the capture moment to the millisecond;
- the millis_lo field is negative (`-950080228`, 10 characters) — bit 31 of uint32(ms) is set;
- prepared = 1337 (`WASafeSignalsPrepare()` was not called in the experiment);
- envelope: 223 base64url characters → 167 B = ephPub 32 + ct 119 + tag 16.

### 7.3. Freshness across runs

Five consecutive native `Create→Get→Release` calls in one process produced five different plaintexts at a stable length of 119 B: base varied (64060 → 23797 → 40002 → ...), as did rand, millis, and the ephemeral key. No field is "frozen" between requests.

## 8. Value conditions and size distribution

Plaintext length statistics (format modeling, 200k trials with realistic field distributions):

| Quantity | Value |
|---|---|
| Ceiling | `1 + 13×5 + 5 + 10 + 10 + 5 + 5 + 18 = 119` characters |
| Mean / median | 116.96 / 118 |
| P(len=119) | 0.48 (mode); P(len=118)=0.38; P(\|len−119\|≤1)=0.86 |
| Bimodality | base ≥ 10000 (p≈0.90) → 117–119; base < 10000 (p≈0.10) → ~99–112 |

The length **does not depend** on content: codes ≤1108 < 4096 never touch bits ≥12 of the value's width (the all-fail/all-success scenarios give the same length distribution); prepared 1337/4242 — the same length. Only the digit count of base (4 vs 5), the length of rand (9–10), and millis_lo (9–10 + a possible minus) move the length. Field-composition argument: 18 fields would give a ceiling ≤113, 20 fields a minimum ≥121; exactly 19 fields land right at the live 119.

**When it is regenerated:** for every /v2/code and /v2/register request (a new ephemeral X25519 pair, fresh gettimeofday/srand/rand, a new base). Only the size matches, never the value. The value is not cached anywhere before sending; the plaintext buffer is wiped (`WASafeSignalsRelease`: memset 0 + free).

**Enablement gates:** WASafe signal collection is gated by AB flags 15 and 16 (values 22925/24048, checked via `JniBridge.jnidispatchIII`); the configuration kill switch is the SharedPreferences `wsafeplatform_context`/`runtime_override` (default 1062737884 = 0x3F57B85C ≈ float 0.8431), read natively via `JniBridge.jnidispatchI(5)`.

## 9. Role for the server's anti-fraud

The server decrypts `em` with the private half of the X25519 pair, reads base, removes the XOR masks (base+i*7, base+91, +98, +105, +112) and obtains:

1. the **format version** (field 1) — scheme compatibility/evolution;
2. the **13 stat results** — a root/emulator detection matrix: success on indices 0/1 = a VM; 3–7, 9–12 = root; 2 = a live Android; 8 = an unusual environment. The expected outcome is fail on the su paths and the VM paths;
3. the **creation time to millisecond precision** (millis lo/hi) — cross-checked against the request reception time: a replay/forwarding of someone else's `_gs` is spotted instantly;
4. **rand** — uniqueness and validation of a "live" generator;
5. **prepared** — whether the client went through the `WASafeSignalsPrepare` phase (4242) before sending, i.e. whether generation followed the standard path rather than a direct call;
6. an **implicit memory attestation**: if the libwasafe string decoder was replaced/poisoned, the path decodes yield `""`, the stat branches fall into fail (+1000), and `WASafeStatus()` returns 3 — a tampered process cannot assemble a valid profile.

From this picture the server makes decisions about captchas, rejections, and registration bans. Faking `_gs` requires all at once: a correct CSV profile of a reference device, fresh timestamps, a correct X25519 envelope under the server key, and not setting off the kill switch inside libwasafe.
