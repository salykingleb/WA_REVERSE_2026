# _gi — encrypted installation attestation block (JSON of 10 fields: dex/lib hashes, firmware, APK)

## Place in the flow

The `_gi` parameter (the gate name from the obfuscated native table, sid 87) is attached to ALL
registration HTTP requests (`/v2/code`, `/v2/register`, makeConsentRequest, pre-chatd AB check,
verifySecurityCode, etc.) in one place:

```
X/IF6.java:553-567 (A0S):
    JniBridge.jvidispatchIOO(7, application, wajContext)          // native state initialization
    map2 = JniBridge.jvidispatchOOO(16, application, wajContext)  // Map<String, ByteArray>
    map.putAll(map2)                                               // _gi, _gg, _ga, _ge, _gp, _gs as a single package
```

The value is formed entirely natively (in the Java/DEX layer there is no `_gi` string — verified by a scan
of the string tables of all classes*.dex); it arrives in Java as a ready base64 string of the encrypted JSON.
A0S call sites: `RequestCodeRepository$requestCode$2.java:402` (/v2/code), `IF6.java:694/804/1417/1566`
(register and the other registration HTTP). Within a single registration the _gi blob in /v2/code and /v2/register
is **byte-for-byte identical** (cache/reuse). After login the same fields (except the _gi wrapper)
go into the XMPP integrity_payload (see _lh.md).

## Wire format (+ live example)

An ASCII JSON of **strictly 10 keys in the EXACT order** `_dh, _iln, _isb, _ip, did, _p, _ln, _ist,
_icr, _is`. The firmware key is **`did` WITHOUT an underscore** (confirmed by the native string table
sid 103 and the live capture; there is NO `_did` string in the table of 106 sids). The `sizeInBytes` analogue `_isb` is a string.

Live plaintext (2026-09-22, WhatsApp 2.26.35.75, identical in code and register):

```json
{"_dh":"bePtUujXdvGCCbQXxkVpyrkEEpqzpWlzP4IGvEe/Dd4=",
 "_iln":"i7ls4vAnG2Bvmsp8NDKh7uREGcY4q8Hi4JtdCLix+Cc=",
 "_isb":"127447992",
 "_ip":"com.whatsapp",
 "did":"TP1A.220624.014.A325FXXSBDXJ1",
 "_p":"/data/app/~~MrDQfLH2…/com.whatsapp-…/base.apk",
 "_ln":"KMr1FDZ5Qv9UsYvUwaPmFmshuABXLq3rfxeELvAebKk=",
 "_ist":"p7OAlmVZ1f6oCEU5cPzBS297HP1rkqpcNIPHHe+ukm4=",
 "_icr":"OKD31QX+GP7GT780Psqq8xDb15k=",
 "_is":"/VEKrB9WsYGAMdFIVQPMo396qeQlYvT2bxZtu1gMPgw="}
```

(in the real plaintext `/` is escaped as `\/`). Then — the shared AES-256-CBC envelope (IV||ct, PKCS7,
base64.Std) + percent-encoding.

## How it is formed in the application

| Field | Formula (proven) | Live 2.26.35.75 |
|---|---|---|
| `_dh` | base64.Std(SHA256(classes.dex from base.apk)) — the "dex hash", computed by the native file-SHA getter and cached | `bePtUujXdvGCCbQXxkVpyrkEEpqzpWlzP4IGvEe/Dd4=` (2.26.37.73: `jjyxvURV8j4F8rILRZttTITKv0JCvJG6F52K32iZ/Ag=`; biz 2.26.36.72: `n3yeZqlbOTzoXXwt6XI6V3BwxDQ3mwB+Uv3ZfmGUpOg=`) |
| `_iln` | base64.Std(SHA256(concat(sorted(base names of ALL regular `*.so` files found RECURSIVELY from ApplicationInfo.dataDir)))) — in practice 79 files of `files/decompressed/libs.spo/`; concatenation WITHOUT a separator, the .so content is NOT hashed | `i7ls4vAnG2Bvmsp8NDKh7uREGcY4q8Hi4JtdCLix+Cc=` (w4b, 84 names: `Mw3rbAdtB9wUHE6w9vpff27e6I9/ruCa555vkJXh+N4=`) |
| `_isb` | size of base.apk in bytes (string) | `"127447992"` (37.73: `"129472282"`) |
| `_ip` | packageName | `com.whatsapp` |
| `did` | the device firmware string (NOT a hash) | `TP1A.220624.014.A325FXXSBDXJ1` |
| `_p` | the base.apk path = ApplicationInfo.sourceDir (the same source as gpia.p, cache global 0x310) | `/data/app/~~MrDQfLH2…/base.apk` |
| `_ln` | base64.Std(SHA256(concat(sorted(top-level names of nativeLibraryDir)))) — NOT recursive, hidden entries included, only `.`/`..` are skipped; on an empty/unreadable directory (extractNativeLibs=false, lib/arm64 empty) — FALLBACK: base64.Std(SHA256("signal_error")) | `KMr1FDZ5Qv9UsYvUwaPmFmshuABXLq3rfxeELvAebKk=` (= the fallback constant; the same for wa and w4b) |
| `_ist` | = gpia.shatr: base64.Std(SHA256(base.apk[0:0xA00000])) — the same first 10 MiB, one computation for both fields, DataStore cache; on error `"-"` | `p7OAlmVZ1f6oCEU5cPzBS297HP1rkqpcNIPHHe+ukm4=` |
| `_icr` | = gpia.cert: base64.Std(SHA-1 of the DER certificate, 822 B) | `OKD31QX+GP7GT780Psqq8xDb15k=` |
| `_is` | = gpia.sha256: base64.Std(SHA256(base.apk as a whole)) | `/VEKrB9WsYGAMdFIVQPMo396qeQlYvT2bxZtu1gMPgw=` |

### The did formula (firmware)

The live value = **Build.ID + "." + Build.VERSION.INCREMENTAL** (Samsung: ID=`TP1A.220624.014`,
INCREMENTAL=`A325FXXSBDXJ1` = the MODEL part + `XXS` + region/year/month). Reconstruction of the value
for a synthetic device profile:

```
did = {Build.ID} . {MODEL_part} {XXS} {insideCode}
      MODEL_part  = the model without the vendor prefix: SM-A325F → A325F
      insideCode  = 4 letters from the set ABCDEFXPG + 1 digit at the tail,
                    the digit is deterministic: SHA256(authkey + Build + Model)
```

Proven discrepancies with real Samsung values: (1) the dot separator between Build.ID and
the model is MANDATORY (`…014.A325F…`, not `…014A325F…`); (2) the real alphabet of the inside code is the full
A–Z (in the live `BDXJ1` there is the letter `J`, which the set ABCDEFXPG does not have — the live value is in principle
not reproducible with this set); (3) the real code is determined by the firmware, not by the authkey. did
is transported as is (from the DataStore/config provider), the application does not compute it.

## Native implementation (libwhatsapp.so, base 0x7cfd633000)

- The builder is the same `__WCAAPIGenerateGPIAParamOnMainImpl` @0x7cfdd3b9a4…0x7cfdd3c068 (builds gpia+`_gg`+`_gi`
  at once; registers x19/x22/x21). The 6 keys "shared" with gpia are written in pairs (addresses in gpia.md);
  _gi has exactly **10** unconditional keys in native — by the count in the reference capture.
- The 4 "own" _gi keys at the tail of the builder: @0x7cfdd3bc84 — the config-provider value (AB channel,
  `0x7cfdd31bfc(ctx,1)`); @0x7cfdd3bdc4 — the getter **0x7cfdd3a5c8** (`_ln`, dir sid 0x2c
  "nativeLibraryDir"); @0x7cfdd3be08 — the getter **0x7cfdd3aa18** (`_iln`, dir sid 0x2b "dataDir",
  inside it the step 0x7cfdd3ab98 — the same traversal routine as in the .so traversal); @0x7cfdd3be44 — the wrapper
  **0x7cfdd3c5e0** → `WAJIntegrityDataStoreCopyValue` (0x7cfdd23ee0) for the shatr/_ist cache.
- `_iln`: the traversal routine `0x7cfdd3ab98(dir, vector, depth)` — opendir/readdir/stat; a regular file whose
  name ends in `.so` → push_back of the BASE name; directories → recursion depth+1; skip of exact `.`/`..`;
  then `0x7cfdd2d898` = std::sort (introsort), `0x7cfdd3a93c` = concatenation (the accumulator string is EMPTY,
  without a separator), SHA256: init `0x7cfdd71df4` (ctx 0x68 B), update 0x7cfdd71e44, final
  `0x7cfdd71e94` (malloc(0x20) → SHA256_Final → base64(out,32)).
- `_ln`: stat(dir)→S_IFDIR, opendir, a NON-recursive collection of all names, the fallback on failure —
  `0x7cfdd31c10(0x41)` = the string sid 0x41 = **"signal_error"** → the same SHA256→base64.
- The native library directory traversal at @0x7cfdd3a000-0x7cfdd3b338 (strings `.`, `..`, `/`, `.so`,
  `failed_sha`, `invalid_library_name`) — computes the .so hashes and puts them into the DataStore.
- The obfuscated name table `0x7cfdd76a64(sid)`: an AES-128-ECB blob, the key = the decoy string+NUL,
  zero-padded to 16 B. Decoded: sid 0="/v2/reg_onboard_abprop", 1="/v2/security",
  0x2b="dataDir", 0x2c="nativeLibraryDir", 0x2d="getExternalFilesDir", 0x41="signal_error".
  The _gi keys: `_dh`=67, `_ln`=69, `_iln`=70, `_isb`=71, `_is`=72, `_icr`=73, `_ip`=77, `_gi`=87,
  `did`=103 (the _ga cluster: ai=94, ap=95, ae=96, mu=97, mp=98, bi=99; nearby aid=102).
- The table DOES contain keys ABSENT from the live 2.26.35.75 capture: `_das, hwfp, _lh, _n, sv, sb,
  _acg, _acj` — do NOT add them to the _gi blob (the appearance of `_lh` in 2.26.37.73 is only in the
  login integrity_payload, not in _gi).

## Value selection conditions

- `_ln` — the constant `KMr1FDZ5Qv9UsYvUwaPmFmshuABXLq3rfxeELvAebKk=` on all recent builds
  (nativeLibraryDir is empty due to extractNativeLibs=false and superpack delivery of the libraries);
  it would differ only on devices with real .so files in nativeLibraryDir.
- `_iln` — a constant of the APK version (the set of .so in libs.spo changes with the version; wa=79, w4b=84 names);
  `.superpack_version` is NOT included in the list (the `*.so` filter).
- `_ist`/`shatr`, `_is`, `_icr`, `_isb`, `_p`, `_ip` — bound to the installed APK (they change with
  the version/package); `_dh` — to the version's classes.dex.
- `did` — from the device firmware; on the same phone it is identical for wa and w4b.
- Composition invariant: exactly 10 keys, the order `_dh,_iln,_isb,_ip,did,_p,_ln,_ist,_icr,_is`.

## Cryptography and encoding

JSON (the order above, `/`→`\/`) → the AES-256-CBC envelope (key SHA256(std-base64(authkey)), IV 16 B
random ‖ PKCS7, base64.Std, `_WAJIntegrityCreateEncryptedAES256CBC` @0x7cfdd3b3a4) → Java
percent-encoding (X/HSJ.A00). The name sorting in _iln/_ln is byte-wise lexicographic
(std::sort over std::string; equivalent to Python `sorted()`).

## Examples

- The full live blob — see "Wire format" (both sessions: 2.26.35.75 and 2.26.37.73, where
  `_dh=jjyxvURV…`, `_isb="129472282"`, `_is=1oUpYgdN…`; `_iln`, `_ln`, `_icr` are byte-for-byte equal to
  the 2.26.35.75 values — stable across versions).
- Recomputing the constants for a new APK version: sha256/shatr/_dh/_iln are computed by the formulas above from
  the corresponding base.apk and the libs.spo listing; no device dump is needed (except for did and _p).
