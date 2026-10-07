# libwasafe.so + libwasafedeps.so — the native engine of the `_gs` parameter

**Application:** WhatsApp Android 2.26.35.75 (versionCode 263507522), arm64-v8a.
**Obtained:** a memory dump of the live com.whatsapp process (Samsung SM-A325F, pid 8643) after a forced lazy load via `JniBridge.jvidispatchIOO(7, ...)`; a complete disassembly pass (1608 instructions of libwasafe + 2489 instructions of libwasafedeps), string-table decoding, 5 runtime runs with interception of the plaintext/GCM key/X25519. The `_gs` format that the library produces is analyzed in `_gs.md`.

---

## 1. Summary

| Component | Size (code) | Role |
|---|---|---|
| `libwasafe.so` | 0x1920 (~6.4 KB) + rodata 0x103c | environment signal collection (13 stat checks), plaintext formatting, obfuscated strings, kill switch |
| `libwasafedeps.so` | 0x26e4 (~10 KB) | the crypto envelope: X25519 (statically built in) + AES-256-GCM (mbedtls from libwhatsappmerged) + base64url |

The libraries are tiny, all the logic has been read in full; the runtime values in .data matched the disassembly analysis.

## 2. Delivery and loading

- Both libraries are **absent from base.apk and the split configs**; they are listed in the super pack manifest `libs.so` (offset ~0x140a: `libwasafe.so … |-libwasafedeps.so`) and downloaded from the server. The names are also present in the Java list of known libraries (`X/C0Ef.java`).
- libwhatsapp.so itself is decompressed from the super pack into `/data/data/com.whatsapp/files/decompressed/libs.spo/` (confirmed by maps.txt); the WASafe pair is dropped there as well.
- Loading is **lazy**: after process start there are NO libwasafe regions in `/proc/<pid>/maps`; they appear after the first `jvidispatchIOO(7, app, wajCtx)` call.
- After the trigger, in maps (pid 8643): `libwasafe.so` — 7bc226a000–7bc2279000 (r/rx/ro2/rw), `libwasafedeps.so` — 7bc2283000–7bc2293000. Dumps of the r/rx/ro2/rw regions of both libraries were taken over these ranges (transport `dd | base64` — a plain dd through Git Bash corrupts the data with CRLF).

The loader in libwhatsapp.so is `EnsureWasafeLoaded()` @ **0x7cfdd2f630**: `dlopen("libwasafe.so", 1)` (string in rodata @ 0x7cfd702c32) → on failure a Java callback `JniBridge.jnidispatchI(4)` → `WhatsAppLibLoader.BR1({"wasafe"})` (jadx: `com/whatsapp/wamsys/JniBridge.java:676`, log tag `WCAAPIEnsureSafeLibraryLoaded`) → retry. Then `dlsym` of the four exports, slots in .bss **0x7cfe391240..0x7cfe391258**:

| Slot | Export |
|---|---|
| +0x00 | `WASafeSignalsCreate` |
| +0x08 | `WASafeSignalsGetBase64EncodedData` |
| +0x10 | `WASafeSignalsRelease` |
| +0x18 | `WASafeSignalsPrepare` |

The export names sit in libwhatsapp.so rodata in plain text.

## 3. The JNI bridge: how the Java side calls WASafe

The full chain (every link confirmed by jadx + disassembly of the dump):

```
RegistrationHttpManager (X/IF6.java:553-567 A0S)
  ├─ JniBridge.jvidispatchIOO(7, application, wajContext)        // native decl JniBridge.java:575
  │    → 0x7cfdd1dc44 → 0x7cfdd2c390 → 0x7cfdd2c404               // гейты AB 15/16 (22925/24048)
  │    → 0x7cfdd2f3e8 (сбор WASafe-сигналов):
  │         signals = WASafeSignalsCreate(0)                      // обёртка 0x7cfdd2f82c
  │         envelope = WASafeSignalsGetBase64EncodedData(signals) // обёртка 0x7cfdd2f87c
  │         native-карта {"em": envelope} → JSON (0x7cfdd2fc10/0x7cfdd42d6c)
  │         → строка {"em":"<b64>"} в wajContext+0x80 → в params под ключом "_gs" (ID 91)
  │         WASafeSignalsRelease(signals)                          // обёртка 0x7cfdd2f8d0
  └─ JniBridge.jvidispatchOOO(16, application, wajContext)        // native decl JniBridge.java:623
       → 0x7cfdd1e930 (jump table @0x7cfd92622d, 18 селекторов)
       → 0x7cfdd2c59c: сборка Map{String,ByteArray} из полей wajContext +0x50…+0x80
         (7 ключей: _gg,_gi,_gp,_ga,_ge,aid,_gs) → putAll в параметры запроса
```

`WASafeSignalsPrepare` is called by a separate wrapper **0x7cfdd2f920** (call region 0x7cfdd2f58c) with an int argument from a property request under a QPL duration measurement — this same native point also serves to deliver the configuration value into the library (`WASafeSignalsConfigure`, the string decoder key; runtime value 0x3f5817dc in .data at 0xebe0).

The Java side contains neither the `_gs` literal nor any generating code — all generation and serialization are native.

## 4. Exported functions of libwasafe.so

Offsets are from the module base (0x7bc226a000 at dump time):

| Export | VA | Size | Semantics (from the disassembly) |
|---|---|---|---|
| `WASafeSignalsConfigure(int)` | **0x6498** | 32 | `*(u32*)0xebe0 = arg` — sets the string decoder key (FNV flag) + **resets the poison flag** 0xebdc to 0 |
| `WASafeSignalsPrepare()` | **0x65b0** | 20 | `*(u32*)0xebd8 = 0x1092` → **4242** — the source of the prepared flag in the 19th field of `_gs` |
| `WASafeStatus()` | **0x65c4** | 32 | `*(u32*)0xebdc != 0 → 3 (tampered)`, otherwise `1 (ok)` |
| `WASafeSignalsCreate(int)` | **0x65e4** | 168 | gate: `arg != 0 → NULL` (a zero argument is the condition of the regular call); plaintext assembly (0x50a0) → encryption (0x64b8) → a handle `malloc(16){char* b64, u32 len}` |
| `WASafeSignalsGetBase64EncodedData(h)` | **0x668c** | 16 | `return h->ptr` — a base64url C string (the envelope) |
| `WASafeSignalsRelease(h)` | **0x669c** | 84 | `memset(buf, 0, len)` **then** `free` — the buffer is wiped before release: the plaintext/envelope cannot be fished out of the heap after Release |

Runtime .data state (shifted in the dumps): `0xebd8 = 1337 (0x539)` — the default prepared before the Prepare call; `0xebdc = 0` — the poison flag is clean; `0xebe0 = 0x3f5817dc` — the decoder key set by Configure.

**Imports of libwasafe.so:** `gettimeofday, srand, rand, stat, malloc, free, __vsnprintf_chk, memcpy, memset` (libc) + 8 `WASafeDepsCrypto*` functions (libwasafedeps). NEEDED: libwasafedeps.so, libc.so.

## 5. libwasafedeps.so — the crypto dependency

Exports (all confirmed in .dynsym from memory):

```
WASafeDepsCryptoInputCreate          WASafeDepsCryptoInputSetServerPublicKey
WASafeDepsCryptoInputSetData         WASafeDepsCryptoInputRelease
WASafeDepsCryptoEncryptBase64        WASafeDepsCryptoResultGetData
WASafeDepsCryptoResultGetLength      WASafeDepsCryptoResultRelease
```

- **NEEDED:** libwhatsappmerged.so, libc.so. Imports `mbedtls_{ctr_drbg, entropy, gcm, platform_zeroize}` and `MCF*/MCI*` helpers (including `MCIBase64EncodeURLSafeCreateASCIIString`) from libwhatsappmerged, pthread.
- **X25519 is NOT imported** — the implementation is built in statically: function **+0x597c** (inputs priv/peer, output shared), curve base point `09 00…00` in rodata @ **0x100a**.
- Encryption (`WASafeDepsCryptoEncryptBase64` @ **0x53f4**): DRBG private key (personalization `"wa_crypto"`) → clamp (`priv[0]&=0xf8; priv[31]&=0x3f; priv[31]|=0x40`) → ephPub → ECDH with the server key `8e8c0f74…2512302d` (32B as a flat blob in libwasafe rodata @0xd50, the same key that encrypts ENC in libwhatsapp) → **raw shared without KDF** into `mbedtls_gcm_setkey(AES, shared, 256)` → `crypt_and_tag(ENCRYPT, IV=12 zeros, AAD=NULL, tag=16)` → `ephPub‖ct‖tag` → base64url without padding → wipe of priv/shared (`platform_zeroize`). The plaintext limit is 0x2800.
- The server key is passed via `WASafeDepsCryptoInputSetServerPublicKey`, the plaintext via `WASafeDepsCryptoInputSetData` (it is exactly this input that was intercepted by a hook to capture the `_gs` plaintext).

## 6. The generation pipeline inside libwasafe (the assembly function at 0x50a0)

1. Decoding of 17 obfuscated strings from rodata (0xc18–0xd50) with the decoder **0x5830(src, total_len, salt=index, dest, out_len)**.
2. 13 calls to the stat helper **0x56e4(path, code)**: `stat(path)==0 ? code : code+1000` (`path==NULL → code+1000`); the base codes are passed as immediates: `[10, 11, 0, 100, 101, 102, 103, 104, 1, 105, 106, 107, 108]`.
3. `gettimeofday` ×2: the first for the seconds/microseconds of `base = ((sec^usec)&0xefff)+0x1000`; the second for `srand(sec2^usec2)` → `rand()` and `millis = sec*1000 + usec/1000`.
4. A `malloc(16){ptr,len}` handle + a `malloc(0xf0)`=240 B buffer.
5. A chain of `__vsnprintf_chk` with the formats `"%d"`, `",%d"`×13, `",%u"`, `",%d"`, `",%d,%d"`, `",%d"` → a CSV of 19 fields (the field formulas are in `_gs.md` §4).
6. Encryption by the 0x64b8 wrapper (→ libwasafedeps EncryptBase64) → the envelope into the handle → `GetBase64EncodedData` hands it to libwhatsapp.

## 7. String obfuscation: an FNV-1a-32 key + a 4-round Feistel network

### 7.1. What is hidden

17 strings in rodata (0xc18–0xd50, interleaved with the server key @0xd50):

| salt (index) | String |
|---|---|
| 0x00 | `/sys/module/vmw_pvscsi` |
| 0x01 | `/sys/module/vboxsf` |
| 0x02 | `/system/build.prop` |
| 0x03 | `/system/app/Superuser.apk` |
| 0x04 | `/sbin/su` |
| 0x05 | `/system/bin/su` |
| 0x06 | `/system/xbin/su` |
| 0x07 | `/data/local/xbin/su` |
| 0x08 | `/proc/version` |
| 0x09 | `/data/local/bin/su` |
| 0x0A | `/system/sd/xbin/su` |
| 0x0B | `/system/bin/failsafe/su` |
| 0x0C | `/data/local/su` |
| 0x0D | `%d` |
| 0x0E | `,%d` |
| 0x0F | `,%u` |
| 0x10 | `,%d,%d` |

The salt of a path **equals the index of the stat check** — the call table at 0x50c8–0x52ac: `(src_va, total, salt, out_len)`, e.g. `(0xC18, 0x18, 0x00, 0x17)` → vmw_pvscsi, `(0xD40, 8, 0x0D, 3)` → `"%d"`. `/system/build.prop` and `/proc/version` live exactly in this table — they are absent from the libwhatsapp.so string table (that one serves `_ge` and holds 11 of the 13 paths in a separate pool).

### 7.2. The decoding algorithm (from the disassembly of the function at 0x5830)

**The round key** is FNV-1a-32 (basis 0x811C9DC5, prime 0x01000193) over 24 bytes:

```
key(flag, salt, round_const, block_idx) = FNV1a32(
    LE32(flag ^ 0x68B0B56B) ‖ LE32(flag ^ 0x900529EE) ‖
    LE32(flag ^ 0xB24E5F12) ‖ LE32(flag ^ 0x16618003) ‖
    LE32(salt) ‖ [round_const,0,0,0, block_idx,0,0,0] )

где flag = *(u32*)0xebe0  (устанавливается WASafeSignalsConfigure; runtime 0x3F5817DC)
```

**Decryption** is a 4-round Feistel network over each 8-byte block (the lo/hi halves are u32 LE):

```
r1 (round_const=3): K = key(...); lo ^= ror32( ((hi ^ K) + 0x5757845D) mod 2^32, 8 )
r2 (round_const=2): K = key(...); hi ^= ror32( ((lo ^ K) + 0x8B7441DA) mod 2^32, 3 )
r3 (round_const=1): K = key(...); lo ^= ror24(hi ^ K)      // ror6→ror7→ror11 композируются в ror24, без аддитивной константы
r4 (round_const=0): K = key(...) ^ 0xF2A2A4ED ^ 0x40F07D3B; hi ^= (lo ^ K)
```

Tail: `dest[min(total_len, out_len)] = 0`; blocks are taken 8 bytes at a time from `src` (the number of blocks = `total_len >> 3`), the output is truncated to `out_len` (including the NUL).

**The deobfuscation method used during the reverse engineering** (reproducible on the dump): take the library image from the dump, read the 8-byte blocks by VA from the call table, compute the 4 FNV-1a-32 round keys for each block from the formula above (flag from .data 0xebe0, captured by the same dump), apply the rounds r1→r4 in the reverse of the stored order — out comes the plaintext string. All 17 strings decode correctly byte-by-byte, which closes the path table.

## 8. Poison / kill switch: why `_gs` attests memory integrity

A self-check is built into the decoder at 0x5830: **printability validation runs on the string with salt==0** (`/sys/module/vmw_pvscsi`). If a non-printable byte appears after decryption (the result of replaced rodata, a patched decoder, or a wrong flag key):

1. the poison flag is set, `*(u32*)0xebdc = 1`;
2. **all subsequent decodes return `""`** — the paths and formats become unavailable, the stat helper receives NULL/empty paths → all checks fall into the fail branch (+1000), the plaintext degenerates;
3. `WASafeStatus()` returns **3 (tampered)** instead of 1 (ok).

Only `WASafeSignalsConfigure(int)` can reset the poison (overwriting the key at 0xebe0 + zeroing 0xebdc) — i.e. the legitimate path with the server config, not a patcher.

The practical meaning: the `_gs` value is not only an environment report but also an **implicit attestation of process integrity**: to assemble a valid profile it is required that (a) the rodata with the strings has not been replaced, (b) the decoder has run with the correct configuration key, and (c) the flags in .data (0xebd8/0xebdc/0xebe0) are in a consistent state. Any interference with libwasafe's memory is reflected in the plaintext (the fail picture) and in the library's status. Additionally, `WASafeSignalsRelease` wipes the buffer before free — neither the plaintext nor the envelope can be pulled out of the process heap after it is freed.

Enablement gates and the configuration kill switch at the wrapper level: AB flags 15/16 (values 22925/24048, `JniBridge.jnidispatchIII`, the map in `X/C0RY.java`); SharedPreferences `wsafeplatform_context`/key `runtime_override` (default 1062737884 = 0x3F57B85C ≈ float 0.8431), read natively via `JniBridge.jnidispatchI(5)` (`JniBridge.java:685`).

## 9. Runtime verification (proofs on the device)

Frida hooks on the live process + direct calls of the exports:

| Proof | Result |
|---|---|
| hook on `WASafeDepsCryptoInputSetData` | the `_gs` plaintext captured — a CSV of 19 fields, 119 B ×5 runs |
| hook on `mbedtls_gcm_setkey` | the GCM key == X25519(priv, server key), 32 B |
| hook on the internal X25519 (+0x597c) | priv/peer/out: the native shared == independently computed |
| envelope decryption | AES-256-GCM(intercepted key, IV=0x00×12, AAD=none) → exactly the intercepted plaintext |
| envelope length | 223 base64url chars → 167 B = ephPub 32 + ct 119 + tag 16 (all runs) |
| `WASafeSignalsCreate(0)→Get→Release` ×5 | different plaintexts/keys/envelopes, identical length — confirmation of FRESH |
| direct calls of `WASafeStatus` | 1 (ok) on a clean process; the poison mechanics confirmed statically |

Artifacts: `runtime_capture.json` (plaintext/GCM key/envelopes ×3), `x25519_debug.json` (priv/peer/out ×4), `map16_result.json` (the selector 16 map); Frida agents `wasafe{,2,3}.js`, drivers `trigger_load.py / runtime_verify2.py / direct_create.py` in `frida_wasafe\`.

## 10. Address summary table

| Object | Address | Library |
|---|---|---|
| WASafeSignalsConfigure / Prepare / Status | 0x6498 / 0x65b0 / 0x65c4 | libwasafe |
| WASafeSignalsCreate / GetBase64EncodedData / Release | 0x65e4 / 0x668c / 0x669c | libwasafe |
| plaintext assembly / stat helper / string decoder | 0x50a0 / 0x56e4 / 0x5830 | libwasafe |
| encryption (the wrapper in the library) | 0x64b8 | libwasafe |
| obfuscated strings / server key | rodata 0xc18–0xd50 / 0xd50 | libwasafe |
| .data: prepared / poison / decoder key | 0xebd8 / 0xebdc / 0xebe0 | libwasafe |
| WASafeDepsCryptoEncryptBase64 | 0x53f4 | libwasafedeps |
| X25519 (static) / base point | 0x597c / 0x100a | libwasafedeps |
| EnsureWasafeLoaded (dlopen+dlsym) | 0x7cfdd2f630 | libwhatsapp (dump VA) |
| signal collection / Create/Get/Release/Prepare wrappers | 0x7cfdd2f3e8 / 0x7cfdd2f82c / 0x7cfdd2f87c / 0x7cfdd2f8d0 / 0x7cfdd2f920 | libwhatsapp |
| selector 16 (the _gs map) / selector 7 (warm-up) | 0x7cfdd2c59c / 0x7cfdd1dc44 | libwhatsapp |
| dlsym export slots | 0x7cfe391240..0x7cfe391258 | libwhatsapp (.bss) |
| insertion of {"em":...} / JSON serialization | 0x7cfdd2fbbc / 0x7cfdd2fc10, 0x7cfdd42d6c | libwhatsapp |
| server key (the copy for ENC) | rodata 0x7cfd9265e0 | libwhatsapp |
