# Flow overview: from the SMS request to account login

WhatsApp Android **2.26.35.75** (versionCode **263507522**). All data comes from reverse
engineering of the app (jadx decompilation of `X.*`, the native libraries `libwhatsapp.so` / `libwasafe.so`)
and live captures from a Samsung SM-A325F phone (firmware TP1A.220624.014.A325FXXSBDXJ1,
root, Wi-Fi without a SIM). Some live facts are from a second session on 2.26.37.73 (263707322).

The flow consists of three major stages. Each parameter has its own README
in subfolders 02–12; this file is a map of the entire path.

```
STAGE 1. SMS REQUEST                    POST https://v.whatsapp.net/v2/code
STAGE 2. REGISTRATION (OTP confirmation) POST https://v.whatsapp.net/v2/register
STAGE 3. ACCOUNT LOGIN                  Chat TCP socket, Noise handshake (WA\x06\x03),
                                        login payload, <success>, then attestations
                                        and initial synchronisation on the same socket
```

---

## Stage 1 — code request: POST /v2/code

Transport: HTTP/1.1 over TLS with a WhatsApp ClientHello profile. Body:
`ENC=<base64url>&H=<base64url>`, Content-Type `application/x-www-form-urlencoded`.
Each parameter is sent **twice**: in plaintext in the URL query and encrypted inside `ENC`.
The `H` header is an ECDSA signature of the ENC string made with a fresh Android Keystore leaf key,
`Authorization` is the DER certificate chain attesting this key.

Parameters (full native order — see `02-transport/params-order.md`):

| Group | Parameters | Where covered |
|---|---|---|
| Phone/locale | `cc`, `in`, `rc`, `lg`, `lc`, `mcc`, `mnc`, `sim_mcc`, `sim_mnc` | 07-locale-operator |
| Identifiers | `id`, `backup_token`, `fdid`, `expid`, `aid`, `pid`, `access_session_id`, `token`, `advertising_id` (may be absent) | 03-identifiers |
| Signal keys | `authkey`, `e_keytype`, `e_ident`, `e_regid`, `e_skey_id`, `e_skey_val`, `e_skey_sig` | 04-signal-keys |
| Integrity (AES-CBC blobs) | `gpia`, `_gi`, `_gg`, `_ga`, `_ge`, `_gp` | 05-integrity |
| Environment security | `_gs` (libwasafe.so) | 06-gs-wasafe |
| Device/network status | `device_ram`, `simnum`, `sim_type`, `hasav`, `network_radio_type`, `cellular_strength`, `roaming_type`, `airplane_mode_type`, `hasinrc`, `db`, `mistyped`, `prefer_sms_over_flash` | 08-device-status |
| UI flow | `method` (sms/voice), `reason`, `entrypoint`, `education_screen_displayed`, `client_metrics`, `recaptcha`, `feo2_query_status` | 08-device-status |

The server response brings the SMS/voice code and `retry_after`-like metadata; the registration
session is considered open (`id`, `backup_token`, `access_session_id`, `token`,
and the entire key system are already created and persistent — see 03/04).

## Stage 2 — registration: POST /v2/register

The same transport and ENC/H/Authorization envelope. Differences from /v2/code (live capture):

- added: `code` (6-digit OTP), `entered="1"`, `has_play_store="true"`,
  `passkey_login_status="not_attempted"`, `sim_operator_name`, `network_operator_name`;
- absent: `method`, `hasav`, `prefer_sms_over_flash`, `token`,
  `education_screen_displayed`, `feo2_query_status`, `reason`;
- FRESH (recreated per request): `_gs`, `_gg`, `gpia`, `_gi`, `client_metrics`;
- CARRIED OVER unchanged from /v2/code: all identifiers, keys, `_ga`, `_gp`, `_ge`, `aid`.

The complete map of app calls (VerifyPhoneNumber → VerifyCodeRepository →
KotlinRegistrationBridge → native wamsys) is in `09-register-flow/call-map.md`.
A successful response creates the account on the server side (the `creation` attribute of the future `<success>`).

## Stage 3 — account login: Noise socket

This is no longer HTTP. A chat TCP socket, a binary Noise protocol:

1. **Preamble** `WA\x06\x03` (+ an optional edge header `ED\x00\x01` with routing_info
   from the previous connection) — folder 10.
2. **Noise handshake** (the first login is always the full `Noise_XX_25519_AESGCM_SHA256`,
   since the server static is not yet saved): ClientHello (ephemeral X25519) →
   ServerHello (server certificate, signed by the issuer `WhatsAppLongTerm1`) →
   ClientFinish (**static = authkey from registration**, encrypted login payload) — folder 10.
3. **ClientPayload** — protobuf with ~25 fields: username (JID), userAgent, connectType,
   lc/oc, etc. — folder 11.
4. **`<success t location pn lid abprops creation>`** — login accepted — folder 10.
5. **The first second after success**: the server itself requests attestations
   (`<safetynet>`, `<gpia>`, ACS `privatestats`), the client uploads prekeys
   (`<iq encrypt>`), the keystore chain, accepts the ToS, receives edge_routing — folder 12.

## Key cross-cutting dependencies

- **authkey** — the single backbone: a registration parameter → the source of the
  AES-256-CBC key for all integrity blobs (`SHA256(std_base64(authkey))`) →
  the Noise handshake static key by which the server recognizes the account at login.
- **e_ident / e_regid / e_skey_*** — registered in /v2/code and /v2/register,
  and uploaded to the server again after `<success>` via `<iq encrypt>`.
- **ENC key** — ephemeral X25519 against the server public key
  `8e8c0f74…2512302d`; the same server key is used by the `_gs` envelope
  (libwasafe) — one WAPUBKEY for the entire registration.
- **JNI dispatcher** `com.whatsapp.wamsys.JniBridge` — the single entry point into
  libwhatsapp.so for integrity parameters and attestations (operation numbers
  jvidispatchIOOOO id=3/4/5 and others — see folder 12).
- **The integrity envelope key** is shared between registration (`gpia`, `_gi`, `_gg`) and
  login (`integrity_payload`, `jws`) — verified by decrypting both live sessions.

## Reference environment of the live captures

| Environment parameter | Value in the capture |
|---|---|
| Phone | Samsung SM-A325F, firmware TP1A.220624.014.A325FXXSBDXJ1 |
| Network | Wi-Fi, no SIM → mcc/mnc/sim_* = 000, sim_type=0, simnum=0 |
| Locale | lg=ru, lc=RU (number cc=7) |
| Number | cc=7, in=9206309125 |
| APK | sha256 /VEKrB9WsYGAMdFIVQPMo396qeQlYvT2bxZtu1gMPgw=, size 127447992, cert OKD31QX+GP7GT780Psqq8xDb15k= |

These values appear in the examples of all READMEs below.
