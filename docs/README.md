# Documentation Table of Contents

A complete guide to the parameters of WhatsApp Android **2.26.35.75** (versionCode **263507522**) recovered through reverse engineering: jadx decompilation (`X.*` classes), the native libraries `libwhatsapp.so` / `libwasafe.so` / `libwasafedeps.so` (memory dumps, disassembler), live captures of `/v2/code`, `/v2/register`, the first XMPP login, and login logs.

It is recommended to start reading with [01-flow-overview.md](01-flow-overview.md).

**Stages:** A — registration (`/v2/code`, `/v2/register`), B — OTP confirmation, C — Noise login to the account, D — after `<success>`.

## Stage A. Registration: POST /v2/code (SMS request) and /v2/register

### [02-transport/](02-transport/README.md) — Transport layer of the /v2/code and /v2/register requests

| Document | Title |
|---|---|
| [`Authorization.md`](02-transport/Authorization.md) | Authorization — header with the keystore attestation certificate chain |
| [`ENC.md`](02-transport/ENC.md) | ENC — the encryption envelope for all registration parameters |
| [`H.md`](02-transport/H.md) | H — request signature (ECDSA-P256-SHA256 over the ENC string) |
| [`http-transport.md`](02-transport/http-transport.md) | transport_HTTPS — the common framework of the HTTP registration requests /v2/code and /v2/register |
| [`params-order.md`](02-transport/params-order.md) | params_order — the exact sequence of query fields for /v2/code and /v2/register |
| [`request_token.md`](02-transport/request_token.md) | request_token — idempotency header of the registration request |

### [03-identifiers/](03-identifiers/README.md) — Identifiers and tokens

| Document | Title |
|---|---|
| [`access_session_id.md`](03-identifiers/access_session_id.md) | access_session_id — registration session UUID v4 → 16 bytes → base64 RawURL |
| [`advertising_id.md`](03-identifiers/advertising_id.md) | advertising_id — Google Advertising ID (GAID), an optional parameter |
| [`aid.md`](03-identifiers/aid.md) | aid — native device identifier: an SHA-256 chain from ANDROID_ID in base64 |
| [`backup_token.md`](03-identifiers/backup_token.md) | backup_token — backup registration token, 20 bytes in SharedPreferences |
| [`expid.md`](03-identifiers/expid.md) | expid — device ID for telemetry, UUID v3/v4 → 16 bytes → base64 RawURL |
| [`fdid.md`](03-identifiers/fdid.md) | fdid — installation phone ID, a UUID v4 string |
| [`id.md`](03-identifiers/id.md) | id — random identifier of a code request attempt, persistent per phone number |
| [`pid.md`](03-identifiers/pid.md) | pid — OS process identifier (Process.myPid) |
| [`request-builder.md`](03-identifiers/request-builder.md) | params_builder — how the /v2/code and /v2/register request parameters are assembled |
| [`token.md`](03-identifiers/token.md) | token — anti-fraud token of the code request: HMAC-SHA1(secretKey, certDER || md5Hash || phone) |

### [04-signal-keys/](04-signal-keys/README.md) — Signal keys (Signal key bundle)

| Document | Title |
|---|---|
| [`authkey.md`](04-signal-keys/authkey.md) | authkey — public key of the client's static X25519 pair (ClientStaticKeyPair) |
| [`e_ident.md`](04-signal-keys/e_ident.md) | e_ident — Signal public identity key (X25519, 32 bytes without prefix) |
| [`e_keytype.md`](04-signal-keys/e_keytype.md) | e_keytype — elliptic curve type of the key bundle (constant 0x05) |
| [`e_regid.md`](04-signal-keys/e_regid.md) | e_regid — Signal protocol registration id (4 bytes big-endian, 1..2147483646) |
| [`e_skey_id.md`](04-signal-keys/e_skey_id.md) | e_skey_id — signed prekey identifier (3 bytes big-endian, 1..16777214) |
| [`e_skey_sig.md`](04-signal-keys/e_skey_sig.md) | e_skey_sig — signature of the signed prekey with the identity key (curve25519sha512, 64 bytes) |
| [`e_skey_val.md`](04-signal-keys/e_skey_val.md) | e_skey_val — signed prekey public key (X25519, 32 bytes without prefix) |
| [`key-system-overview.md`](04-signal-keys/key-system-overview.md) | Generation and storage of the key system during the first registration (summary) |

### [05-integrity/](05-integrity/README.md) — Encrypted integrity parameters

| Document | Title |
|---|---|
| [`_ga.md`](05-integrity/_ga.md) | _ga — block of installation ages and anti-monkey flags (plaintext JSON {bi, ap, ai, mp, ae, mu}) |
| [`_ge.md`](05-integrity/_ge.md) | _ge — environment flags (plaintext JSON {"sb":false,"sv":false}) |
| [`_gg.md`](05-integrity/_gg.md) | _gg — encrypted block with the Google Play Integrity result (JSON {_ic, _it}: code + token) |
| [`_gi.md`](05-integrity/_gi.md) | _gi — encrypted block of installation attestation (JSON of 10 fields: dex/lib hashes, firmware, APK) |
| [`_gp.md`](05-integrity/_gp.md) | _gp — hash of the manifest permission set (SHA256 of the sorted concatenation of 85 permissions) |
| [`_lh.md`](05-integrity/_lh.md) | _lh — hash of the native library from nativeLibraryDir (new field in 2.26.37.73, login integrity_payload) |
| [`encryption-key.md`](05-integrity/encryption-key.md) | Integrity blob encryption key — the shared AES-256-CBC envelope for gpia / _gi / _gg / bi / integrity_payload / jws |
| [`gpia.md`](05-integrity/gpia.md) | gpia — encrypted block of Google Play Integrity APK attestation (JSON of 8 fields about the APK and the token) |

### [06-gs-wasafe/](06-gs-wasafe/README.md) — The _gs parameter and the native library libwasafe

| Document | Title |
|---|---|
| [`_gs.md`](06-gs-wasafe/_gs.md) | The `_gs` parameter — environment security signals (WASafe) |
| [`libwasafe.md`](06-gs-wasafe/libwasafe.md) | libwasafe.so + libwasafedeps.so — the native engine of the `_gs` parameter |

### [07-locale-operator/](07-locale-operator/README.md) — Locale and operator

| Document | Title |
|---|---|
| [`cc-in.md`](07-locale-operator/cc-in.md) | cc / in — country code and national phone number |
| [`lg-lc.md`](07-locale-operator/lg-lc.md) | lg / lc — language and country of the app interface locale |
| [`mcc-mnc-sim.md`](07-locale-operator/mcc-mnc-sim.md) | mcc / mnc / sim_mcc / sim_mnc — network and SIM operator codes |
| [`operator-names.md`](07-locale-operator/operator-names.md) | network_operator_name / sim_operator_name — network and SIM operator names |
| [`rc.md`](07-locale-operator/rc.md) | rc — release channel |

### [08-device-status/](08-device-status/README.md) — Device, network, and UI flow status

| Document | Title |
|---|---|
| [`cellular_strength.md`](08-device-status/cellular_strength.md) | cellular_strength — cellular network signal level |
| [`client_metrics.md`](08-device-status/client_metrics.md) | client_metrics — request JSON metrics (attempts, installation source, SIM indicators) |
| [`device_ram.md`](08-device-status/device_ram.md) | device_ram — device RAM amount in gibibytes |
| [`education-fields.md`](08-device-status/education-fields.md) | education_screen_displayed, clicked_education_link, prefer_sms_over_flash — flash-call education state |
| [`entrypoint.md`](08-device-status/entrypoint.md) | entrypoint — where registration was launched from ("suma" | "create_paa" | absent) |
| [`feo2_query_status.md`](08-device-status/feo2_query_status.md) | feo2_query_status — status of the FEO2 client capabilities request ("client_capabilities_cached" | "success_get_client_capabilities" | …) |
| [`hasav.md`](08-device-status/hasav.md) | hasav — availability of automatic SMS verification (Google Play Services SMS Retriever) |
| [`hasinrc-db.md`](08-device-status/hasinrc-db.md) | hasinrc, db — local registration state (rc2 storage) and developer mode flag (db) |
| [`method-reason.md`](08-device-status/method-reason.md) | method, reason — OTP delivery method and reason for re-requesting the code |
| [`mistyped.md`](08-device-status/mistyped.md) | mistyped — code of the phone number entry method on the registration screen |
| [`network_radio_type.md`](08-device-status/network_radio_type.md) | network_radio_type — active network connection type (radio interface) |
| [`recaptcha.md`](08-device-status/recaptcha.md) | recaptcha — reCAPTCHA flow JSON state ({"stage":"ABPROP_DISABLED"}) |
| [`roaming-airplane.md`](08-device-status/roaming-airplane.md) | roaming_type, airplane_mode_type — roaming and airplane mode |
| [`simnum-simtype-simstate.md`](08-device-status/simnum-simtype-simstate.md) | simnum, sim_type, sim_state, read_phone_permission_granted — SIM card state and phone permissions |

## Stage B. OTP confirmation: /v2/register

### [09-register-flow/](09-register-flow/README.md) — The /v2/register flow (OTP confirmation)

| Document | Title |
|---|---|
| [`call-map.md`](09-register-flow/call-map.md) | POST /v2/register (OTP verification flow) — complete map of calls and parameters |
| [`code.md`](09-register-flow/code.md) | code — 6-digit OTP code for number confirmation |
| [`entered.md`](09-register-flow/entered.md) | entered — numeric code of the OTP entry method |
| [`extra-params.md`](09-register-flow/extra-params.md) | Additional conditional /v2/register parameters: fid, preloads_*, cred_token, old_phone_number, context, security_code (2FA) |
| [`has_play_store.md`](09-register-flow/has_play_store.md) | has_play_store — indicator of an installed Google Play Store |
| [`passkey_login_status.md`](09-register-flow/passkey_login_status.md) | passkey_login_status — stage of the passkey login attempt (wireToken) |
| [`tos_version.md`](09-register-flow/tos_version.md) | tos_version — version of the accepted terms of service ("5") |
| [`vname.md`](09-register-flow/vname.md) | vname — VerifiedNameCertificate (verified business name certificate) |

## Stage C. Account login: chat Noise socket

### [10-login-handshake/](10-login-handshake/README.md) — Account login: Noise handshake

| Document | Title |
|---|---|
| [`client-finish.md`](10-login-handshake/client-finish.md) | ClientFinish: client static key and login payload |
| [`client-hello.md`](10-login-handshake/client-hello.md) | ClientHello: protobuf, state machine, full and resume variants |
| [`connect-preconditions.md`](10-login-handshake/connect-preconditions.md) | Connection preconditions: when the chat socket is opened at all |
| [`noise-modes.md`](10-login-handshake/noise-modes.md) | Noise modes: the full/resume fork, pattern names, PQ variants |
| [`preamble.md`](10-login-handshake/preamble.md) | Preamble: bytes on the socket before ClientHello |
| [`server-hello-certificate.md`](10-login-handshake/server-hello-certificate.md) | ServerHello: DH operations, PQ TLV, and version 6 certificate verification |
| [`success-failure.md`](10-login-handshake/success-failure.md) | Login response: `<success>` and `<failure>`, client-side disconnects before the response |

### [11-client-payload/](11-client-payload/README.md) — Login payload ClientPayload

| Document | Title |
|---|---|
| [`connect-fields.md`](11-client-payload/connect-fields.md) | ClientPayload connection fields: 10, 12, 13, 15, 16, 43 |
| [`device-classes.md`](11-client-payload/device-classes.md) | Fields 36 yearClass and 37 memClass — device performance classes |
| [`lc-oc.md`](11-client-payload/lc-oc.md) | Field 24 lc (successful login counter) and field 23 oc (signature verification) |
| [`misc-fields.md`](11-client-payload/misc-fields.md) | Other ClientPayload fields: 41, 40, 45/46, 47, 18 |
| [`pushname-sessionid.md`](11-client-payload/pushname-sessionid.md) | Fields 7 pushName and 9 sessionId |
| [`structure.md`](11-client-payload/structure.md) | ClientPayload: how the encrypted login payload is assembled |
| [`user-agent.md`](11-client-payload/user-agent.md) | Field 5 userAgent (X/C1Ht) — complete field map |
| [`username-passive.md`](11-client-payload/username-passive.md) | Fields 1 username and 3 passive |

## Stage D. After <success>: attestations and initial synchronization

### [12-post-login/](12-post-login/README.md) — After <success>: attestations and initial synchronization

| Document | Title |
|---|---|
| [`acs-privatestats.md`](12-post-login/acs-privatestats.md) | ACS `privatestats`: blind signatures version 1 and version 2 |
| [`gpia-jws.md`](12-post-login/gpia-jws.md) | Play Integrity after login: `<gpia><request nonce>` → `<ib><gpia><jws>` |
| [`initial-sync.md`](12-post-login/initial-sync.md) | Initial synchronization after success: all remaining nodes |
| [`integrity-payload.md`](12-post-login/integrity-payload.md) | SafetyNet envelope: `<ib><integrity_payload>` (768 bytes) |
| [`keystore-attestation.md`](12-post-login/keystore-attestation.md) | Keystore attestation: `<ib><keystore_attestation>` |
| [`prekeys-encrypt.md`](12-post-login/prekeys-encrypt.md) | Prekey upload: `<iq xmlns='encrypt' type='set'>` immediately after success |
| [`timeline.md`](12-post-login/timeline.md) | Timeline of the first second after `<success>` |
