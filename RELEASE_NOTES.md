# Release notes — 2.26.44.90 update

This update extends the documentation from **2.26.35.75 (263507522)** to
**2.26.44.90 (264409011)**, keeping the original material intact. Facts that changed are
marked in the affected documents and collected in the delta; the mainline documents still
describe 2.26.35.75.

## Added

- **`docs/13-delta-2.26.44.90.md`** — full delta between 2.26.35.75 and 2.26.44.90:
  registration constants, login handshake, ClientPayload, keystore attestation, native
  libraries, `_lh`.
- **`docs/14-post-login-capture-2.26.44.90.md`** — order of the post-login nodes of a fresh
  registration login, with the key data of each node.
- **`docs/15-post-login-chain-2.26.44.90.md`** — per-parameter analysis of every node of the
  post-login chain: builder class and method, value origin, code, AB gates, conditions of
  absence.
- **`docs/12-post-login/keystore-attestation-2.26.44.90.md`** — attestation builder and
  sender in 2.26.44.90, AB 1934/1935/2067/2074/2078/2079.
- **`docs/08-device-status/first-api-level.md`** — the new registration parameter
  `first_api_level` (`ro.product.first_api_level`, AB 39257).
- **`VERSIONS.md`** — versions covered and which facts apply to which build.

## Updated

### Registration

| Constant | 2.26.35.75 | 2.26.44.90 |
|---|---|---|
| `APKversion` / `VersionCode` | 2.26.35.75 / 263507522 | 2.26.44.90 / 264409011 |
| `APKversionBiz` / `VersionCodeBiz` | 2.26.37.72 | 2.26.44.74 / 264407400 |
| `SHA256Personal` | — | `MSherFwxGVrFW62d/1bbwIsDxx4PlT/5kW9JHhRfY4I=` |
| `ShatrPersonal` | — | `zgl2f9Li9IrRlY6RydXO0EmPm6WMfTdvsyrosOszSOs=` |
| `SizeBytesPersonal` | 144 362 968 | 147 363 498 |
| `GIILn` | 42 / 43 names | 44 / 45 names |
| `DexDigestB64` | `fC5+DZsS…` | `QuTjjfD8lrByE713Mf2ceQ==` |
| `MacKeyB64` | `sdvJhddp…` | unchanged |
| `ReleaseDateUnix` | 1794096000 | 1794614400 |
| `CertPersonal` / `ApkDigest` | `OKD31…` / `OYfQQ9…` | unchanged |

- `_gp`: 91 permissions per package; the personal and business sets now differ
  (`IUm79ogL4VaKd/pq5Hdj/5TxErdbGLuSRp3VbtEHg90=` vs
  `69poSQ9I+Je5SQUP3zp74LEklzjVFZfyc5aUVM5iGkc=`). Added `BLUETOOTH` and
  `CREDENTIAL_MANAGER_QUERY_CANDIDATE_CREDENTIALS`; removed the Amazon messaging
  permissions.
- New parameter `first_api_level` (AB 39257, value `ro.product.first_api_level`, omitted
  when the property is absent).
- Transport (`ENC`/`H`, parameter order, TLS) is unchanged.

### Login handshake

- Preamble `57 41 06 03` is unchanged; it is now written by the state machine.
- The v5 branch (`57 41 05 03`), the v5 certificate messages and the
  `noise certificate expired` error were removed.
- The certificate key mismatch error now ends with `; issuer=<issuerSerial>`.
- Noise patterns, `pqMode` values, the v6 certificate and ClientFinish are unchanged.

### ClientPayload

- 38 field numbers instead of 36: added `messageSts` (49) and `integrationAttestation` (50).
- Numbers 1–48 are unchanged; `appVersion = 2.26.44.90`.
- A live payload of a fresh registration is included field by field; fields 49 and 50 were
  not populated in that session.
- Corrected earlier statements: `lidDbMigrated` uses `A04 != null`, `paaLink` is populated,
  `deviceType` gained `SMART_DISPLAY` and `SMART_SPEAKER`.

### keystore_attestation

- The builder (`X/17x`) and the sender (`X/2Eu`) are separate in 2.26.44.90.
- Sender gate AB 1935, throttling AB 2067 with the pref `ka_stanza_sent_ts`.
- Challenge `uint64BE(seconds) ‖ 0x1F ‖ p2`; the live `p2` is 32 bytes supplied by the
  caller, the `SecureRandom(AB 2078)` branch is not used on that path.
- Live chain length 3420 bytes; the node is sent right after `unified_session`.
- AB 2078 and 2079 are still read (through `A0Z`), contrary to an earlier note.

### Post-login traffic

Documented for the first time:

- the order of the post-login nodes (push config, `unified_session`, media connection,
  ACS v1, ToS, prekey upload, attestation, integrity, gpia, account bootstrap, presence,
  sync, stats, comms, ACS v2);
- 22 distinct `xmlns` families, including `usync` contact sync, account crypto, backup
  tokens, profile picture, passive and disappearing-mode queries;
- the prekey upload content (identity key, registration id, 812 one-time prekeys, signed
  prekey and its 64-byte XEdDSA signature);
- the ACS v1/v2 request parameters and their sources;
- `w:stats` built natively from a WAMSys buffer, `w:mex` `flow_id`/`query_id`, QP telemetry
  events, and the fact that `ent:silent_nonce` is an inbound notification type with no
  client builder.

### Native libraries

- New `librust.so`: `smax_rs_stanza`, Shortcake/ML-KEM-768, OxidizedCurve25519.
- `libwhatsapp.so` shrank by 39 % (the handshake code moved out), `libsuperpack.so` is
  byte-identical, `libwhatsappmerged.so` gained `mldsa65`/`mldsa87`, labyrinth seals and
  Tigon MNS/QUIC Noise classes.

### `_lh`

- Formula restated: `base64(SHA-256(content of <nativeLibraryDir>/<name>.so))`.
- `nativeLibraryDir` is empty in 2.26.44.90; candidate hashes for the live libraries are
  listed for matching against a decrypted payload.

## Compatibility

- All existing documents remain valid for 2.26.35.75.
- Where a fact changed, the document carries a "2.26.44.90" note pointing to the delta.
