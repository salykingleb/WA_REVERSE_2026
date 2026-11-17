# Delta: 2.26.35.75 → 2.26.44.90 (registration, login, post-login)

Re-verification of this documentation against **2.26.44.90 (versionCode 264409011)**,
pulled from a live device (SM-A325F, Android 13) and analysed statically (jadx, smali,
native libraries) and dynamically (Frida).

| Artifact | Location |
|---|---|
| APK | `apk_update/pers_2.26.44.90_base.apk` (147 363 498 B) |
| jadx tree | `jadx_264409011/sources` (83 889 classes, 234 decompilation failures) |
| smali | `smali1` (classes.dex), `smali2` (classes2.dex) |
| Native libs | `natives_new/` + `natives_new/decompressed/` (runtime superpack) |
| Live capture | `hooks/wa_agent_log_registration.txt` (registration, 5806 lines) |
| Frida agent | `hooks/agent/agent.ts` (compiled with `frida-java-bridge`) |

## 1. Registration

The registration flow (`/v2/code`, `/v2/register`, `ENC`+`H`, TLS) is **unchanged**.
Only the version-dependent constants were refreshed.

| Constant | 2.26.35.75 | 2.26.44.90 |
|---|---|---|
| `APKversion` / `VersionCode` | 2.26.35.75 / 263507522 | **2.26.44.90 / 264409011** |
| `APKversionBiz` / `VersionCodeBiz` | 2.26.37.72 | **2.26.44.74 / 264407400** |
| `SHA256Personal` (b64 SHA-256 of base.apk) | — | `MSherFwxGVrFW62d/1bbwIsDxx4PlT/5kW9JHhRfY4I=` |
| `ShatrPersonal` (first 10 MiB) | — | `zgl2f9Li9IrRlY6RydXO0EmPm6WMfTdvsyrosOszSOs=` |
| `SizeBytesPersonal` | 144 362 968 | **147 363 498** |
| `SHA256Biz` / `ShatrBiz` / `SizeBytesBiz` | — | `tTZAqKmY…` / `doWAtTzf…` / 166 679 260 |
| `GIDh` (b64 SHA-256 classes.dex) | — | `Yvv3yGdl…` (personal), `fEsWXeLb…` (biz) |
| `GIILn` (b64 SHA-256 sorted lib names) | 42 / 43 names | **44 / 45 names** |
| `HashKey` (b64 MD5 classes.dex) | — | `QuTjjfD8…` (personal), `T+1+lIVX…` (biz) |
| `DexDigestB64` (regtoken) | `fC5+DZsS…` | `QuTjjfD8lrByE713Mf2ceQ==` |
| `MacKeyB64` (PBKDF2 over `about_logo.png`) | `sdvJhddp…` | **unchanged** |
| `ReleaseDateUnix` | 1794096000 (2026-11-08) | **1794614400 (2026-11-14)** |
| `CertPersonal` / `ApkDigest` | `OKD31…` / `OYfQQ9…` | **unchanged** (same signing key) |

### 1.1. Permissions and `_gp`

`_gp = base64(sha256(concat(sorted(manifest permissions))))` — the formula is unchanged,
but the permission set changed, and it now **differs between personal and business**:

| Package | Permissions | `_gp` |
|---|---|---|
| `com.whatsapp` | 91 | `IUm79ogL4VaKd/pq5Hdj/5TxErdbGLuSRp3VbtEHg90=` |
| `com.whatsapp.w4b` | 91 | `69poSQ9I+Je5SQUP3zp74LEklzjVFZfyc5aUVM5iGkc=` |

Changes versus 2.26.35.75: `android.permission.BLUETOOTH` and
`android.permission.CREDENTIAL_MANAGER_QUERY_CANDIDATE_CREDENTIALS` were added;
`com.amazon.device.messaging.permission.RECEIVE` and
`com.whatsapp.permission.RECEIVE_ADM_MESSAGE` were removed. The personal/business
sets differ in `RECEIVE_SMS`, `*.DYNAMIC_RECEIVER_NOT_EXPORTED_PERMISSION`,
`*.permission.BROADCAST`, `*.permission.MAPS_RECEIVE`.

### 1.2. New parameter: `first_api_level`

2.26.44.90 adds `first_api_level` to the registration parameter map (absent in
2.26.35.75). Value = the system property `ro.product.first_api_level`
(`X/AbstractC43828JXg.A00()`), a decimal integer; the key is omitted when the property is
missing/0, and the whole parameter is gated by AB 39257 (`X/K8v.A0R`). Live value on the
test device: `30`. Details: [08-device-status/first-api-level.md](08-device-status/first-api-level.md).

### 1.3. Runtime class names (jadx vs ART)

jadx prefixes obfuscated names with `C<id>`; at runtime the names are the original ones:

| Purpose | jadx | runtime |
|---|---|---|
| Stanza decoder (`readNode`) | `C242314q` | **`X.14q`** |
| Stanza encoder (`writeNode`) | `C41301sR` | **`X.1sR`** |
| ClientPayload provider | `C39641oJ` | **`X.1oJ`** |
| ClientPayload message | `C39651oO` | **`X.1oO`** |
| Keystore attestation builder | `C250617x` | **`X.17x`** |
| Attestation sender | `RunnableC49222Eu` | **`X.2Eu`** |
| `ProtocolTreeNode` | `C0T8` | `X.0T8` |
| Node attribute | `C0T5` | `X.0T5` |

## 2. Login handshake

| Element | 2.26.35.75 | 2.26.44.90 |
|---|---|---|
| Preamble | `57 41 06 03` | **unchanged**, but written by the state machine `X.2DR.run()` |
| v5 branch (`57 41 05 03`) | present but unreachable | **removed entirely** (no version field, no log, no v5 cert protobufs) |
| Framing / `GOA` / 16 MiB limit | — | unchanged |
| Noise patterns, `pqMode` enum, PQ-TLV, ML-KEM-512 | — | unchanged |
| Server certificate v6 (`BW5`/`BW4`/`Baq`), `WhatsAppLongTerm1` | — | descriptors byte-identical |
| v5 certificate (`BW6`/`Bar`), `noise certificate expired` | present | **removed** |
| Cert key mismatch error | plain text | text now ends with `; issuer=<issuerSerial>` |
| ClientFinish | — | unchanged |

## 3. ClientPayload

Message `X.1oO` now has **38 field numbers**; numbers 1–48 are unchanged, two fields were added:

| # | Field | Type | Notes |
|---|---|---|---|
| 49 | `messageSts` | int64 | from `X.1oI.A09` (`duplexMessageStsMicros`) |
| 50 | `integrationAttestation` | message `X.C8t7{staticUnbound=1, onlineSigned=3}` | no Java setter found; filled outside Java |

Live capture of a fresh registration (222 B payload):

```
1  username            = 222583156932850
3  passive             = 1
5  userAgent           = appVersion 2.26.44.90, mcc 250 / mnc 039, os 13,
                         manufacturer samsung, model a32, lg=ru lc=RU,
                         osVersion TP1A.220624.014.A325FXXSBDXJ1
9  phoneId              (fixed32)
12 connectType          = 1
13 connectReason        = 6
23 oc                   = 1
36 memClass             = 2015
40 trafficAnonymization = 0
43 connectionSequenceInfo (fixed32)
44 paaLink              = 0
```

**Fields 49 and 50 were absent in the live payload** — they are optional and were not
populated for this account/session.

Documentation corrections found during the comparison:

- `lidDbMigrated` (41) is set when `A04 != null` — the docs previously said `== null`.
- `paaLink` (44) *is* populated (was documented as "not populated").
- `deviceType` enum gained `SMART_DISPLAY(5)` and `SMART_SPEAKER(6)`.

## 4. keystore_attestation

Builder and sender were split in 2.26.44.90; see
[keystore-attestation-2.26.44.90.md](12-post-login/keystore-attestation-2.26.44.90.md) for details.

- Builder: `X.17x`; gates AB **1934**, alias AB **2074**, random length AB **2078**, key window AB **2079**.
- Sender: `X.2Eu`; gate AB **1935**, throttle AB **2067** with pref `ka_stanza_sent_ts`.
- Challenge: `uint64BE(seconds) ‖ 0x1F ‖ p2`; live `p2` is **32 bytes supplied by the caller**
  (the `SecureRandom(AB 2078)` path is not used on this route).
- Live certificate chain: **3420 bytes** DER, sent as `<ib><keystore_attestation>` right after
  `<ib><unified_session>`.

## 5. Post-login traffic

A full live capture of the first login after registration is documented in
[14-post-login-capture-2.26.44.90.md](14-post-login-capture-2.26.44.90.md): node order,
attributes, blob sizes and the set of features activated after `<success>`.

## 6. Native libraries

| Library | 2.26.35.75 | 2.26.44.90 | Change |
|---|---|---|---|
| `libs.so` (superpack) | 13 544 024 | 14 352 736 | +808 712 B |
| `libsuperpack.so` | 220 920 | 220 920 | byte-identical |
| `libwhatsapp.so` | 14 019 405 (dump) | 8 383 928 | **−39 %** (Shortcake/ML-KEM-768 moved out) |
| `librust.so` | absent | 5 096 368 | **new**: `smax_rs_stanza`, `shortcake`, ML-KEM-768, OxidizedCurve25519 |
| `libwhatsappmerged.so` | 8 256 680 | 6 963 952 | + `mldsa65`/`mldsa87`, labyrinth seals, Tigon MNS/QUIC Noise |

## 7. `_lh`

Formula (unchanged, from `05-integrity/_lh.md`):

```
_lh = base64_std( SHA-256( content of <nativeLibraryDir>/<name>.so ) )
```

New facts for 2.26.44.90:

- `nativeLibraryDir` is **empty** on the device (all libraries are delivered through the
  superpack into `/data/data/com.whatsapp/files/decompressed/libs.spo/`), so the hashed
  file is not one of the statically delivered `.so` files.
- Candidate values `base64(SHA-256(content))` for the live libraries of this build:

| File | Size | b64(SHA-256) |
|---|---|---|
| `libs.so` | 14 352 736 | `aMTANPDS0EcApKdrlS8UtfprGt3/C1dYefbiYKFxvYQ=` |
| `libsuperpack.so` | 220 920 | `/iTYCR1/yW4LPe6c2j1+y17Fy+RYK4bZxCAQsE2Y5F8=` |
| `libunwindstack_binary.so` | 11 800 | `SzFuNPvhZd5A9PS/IQdQvSYLFYm+jWOueGHwst4HB2w=` |
| `libwhatsapp.so` | 8 383 928 | `4ySx1nCRZmHIIOb5iSSVA0quBT/MIJlyqOQg9FL8gbY=` |
| `libwhatsappmerged.so` | 6 963 952 | `C/GQ+b7wlhHckCq1SsgqG6NLJZm6rHc8T/zLwNtBXOM=` |
| `librust.so` | 5 096 368 | `deixXA8RXOIQpBeadohQqiuxBF/jVGbSHAJ9MQtbo1o=` |
| `libcurve25519.so` | 99 464 | `q31b65iWZ0bqE6GOjndQ73yC4p4cvV9skhzFuD7cnFk=` |

Verification plan: hook `mbedtls_sha256_update` / `mbedtls_sha256_finish`
(exported by `libwhatsappmerged.so`) while `integrity_payload` is being built —
that yields the hashed input size and the final digest.

## 8. Still unknown

| Item | Why | How to close |
|---|---|---|
| `_lh` value for 2.26.44.90 | blob is AES-256-CBC encrypted; authkey not captured | `mbedtls_sha256_*` hooks, or decrypt with authkey |
| Field 50 filler | no Java setter | native analysis / live payload with AB enabled |
| `att-stanza-custom` response format | body not decompiled | Frida hook on `X.17x.A04` (already logs `p2`) |
