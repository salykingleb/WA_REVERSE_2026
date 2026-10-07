# WhatsApp: Complete Registration & Login Scheme (Reverse-Engineered Documentation)

Results of reverse engineering the **WhatsApp Android 2.26.35.75** client
(versionCode **263507522**): a complete breakdown of every parameter and
cryptographic mechanism — from requesting the SMS code to the first account
login and the server-side attestations that follow it.

The material was reconstructed from a decompilation of the application
(jadx, obfuscated `X.*` classes), analysis of the native libraries
(`libwhatsapp.so`, `libwasafe.so`, `libwasafedeps.so` — memory dumps,
disassembly) and live traffic captures from a real device (Samsung SM-A325F,
firmware TP1A.220624.014.A325FXXSBDXJ1, Wi-Fi without SIM, lg=ru / lc=RU,
cc=7). Selected facts were confirmed by a second live session on
2.26.37.73 (263707322).

> ⚠️ **Disclaimer.** This material is published for research and educational
> purposes. This project is not affiliated with WhatsApp / Meta. The
> documentation describes how the existing client's protocol works; using it
> to bypass protections, automate mass registrations or violate the ToS is
> outside the goals of this research and remains the sole responsibility of
> the violator.

---

## What is covered

```
STAGE A. POST https://v.whatsapp.net/v2/code        — request the SMS/voice code
STAGE B. POST https://v.whatsapp.net/v2/register    — confirm the OTP, create the account
STAGE C. Chat TCP socket, Noise handshake (WA\x06\x03) — login: ClientHello → server
         certificate → ClientFinish (login payload) → <success>
STAGE D. The first second after <success>           — prekeys, keystore attestation,
         SafetyNet integrity_payload, Play Integrity jws, ACS, synchronisation
```

## Repository structure

```
.
├── README.md                       ← you are here
├── .gitignore
└── docs/
    ├── README.md                   ← FULL TABLE OF CONTENTS (all 83 documents)
    ├── 01-flow-overview.md         ← FLOW OVERVIEW: start reading here
    ├── 02-transport/               ← request envelope: ENC, H, Authorization, request_token,
    │                                  the 50/49 parameter order, TLS
    ├── 03-identifiers/             ← id, backup_token, fdid, expid, aid, pid,
    │                                  advertising_id, access_session_id, token, the builder
    ├── 04-signal-keys/             ← authkey, e_keytype, e_ident, e_regid, e_skey_*
    ├── 05-integrity/               ← gpia, _gi, _gg, _ga, _ge, _gp, _lh + the envelope key
    ├── 06-gs-wasafe/               ← _gs (root/emulator detection) + libwasafe
    ├── 07-locale-operator/         ← cc/in, rc, lg/lc, mcc/mnc/sim_*, operator names
    ├── 08-device-status/           ← device_ram, hasav, network_radio_type, mistyped,
    │                                  method/reason, entrypoint, client_metrics, recaptcha…
    ├── 09-register-flow/           ← /v2/register call map, code, entered, vname,
    │                                  passkey_login_status, tos_version, 2FA
    ├── 10-login-handshake/         ← preamble, Noise XX/IK/PQ, ClientHello/ServerHello/
    │                                  ClientFinish, success/failure
    ├── 11-client-payload/          ← all 48 protobuf fields of the login payload
    └── 12-post-login/              ← post-success timeline, prekeys, keystore attestation,
                                       integrity_payload, gpia jws, ACS, initial sync
```

Every folder has its own `README.md` index; every parameter is described in a
dedicated file following a single template:

1. **Place in the flow** — which request/stage the parameter belongs to;
2. **Wire format** — type, encoding, length + a live example from a capture;
3. **How the app builds it** — Java/Kotlin: class `file:line`, code snippets, persistence;
4. **Native implementation** — `_WCAPI*` / `WASafe*` functions, VA addresses, algorithms;
5. **Value selection conditions** — variant tables, ranges, when it is absent;
6. **Cryptography and encoding**;
7. **Examples**.

## Key cross-cutting facts (tl;dr)

- **`authkey` is the backbone of the whole scheme**: a registration parameter →
  the source of the AES-256-CBC key for all integrity blobs
  (`SHA256(std_base64(authkey))`) → the client's Noise static key by which the
  server recognises the account at login.
- Request parameters are sent **twice**: in the clear in the URL query and
  encrypted inside `ENC` (ephemeral X25519 + server key `8e8c0f74…2512302d` →
  AES-256-GCM, IV = 12 zero bytes). The `_gs` envelope uses the same server key.
- `H` is an ECDSA-P256-SHA256 signature over SHA256(ENC) with a fresh Android
  Keystore leaf key; `Authorization` carries the DER chain attesting it; the
  challenge embeds the request second.
- The first login is always a full `Noise_XX_25519_AESGCM_SHA256` (the server
  static is not saved yet); the login travels in `ClientFinish.payload`, while
  the prekeys go in a separate `<iq xmlns='encrypt'>` only after `<success>`.
- Right after `<success>` the server itself demands attestations: the
  SafetyNet blob is assembled locally (~35 ms, no Google round-trip), the
  Play Integrity jws is the only network call (~4.5 s), plus ACS VOPRF
  receipts.

## Reference live values (SM-A325F session, 2026-09-22)

| Parameter | Value |
|---|---|
| authkey | `rsgyL8y0zyOXIJT4KofOJGJaxrlKUp6TuBsPvGzZAzs` |
| integrity AES key | `SHA256(std_b64(authkey))` = `R41B51Od02uXyjkarGGxbrDvQghVHzMkl9W+HOQDJK4=` |
| e_keytype / e_regid / e_skey_id | `BQ` / `extZ9w` (2065390071) / `E-mO` (1304974) |
| expid | `AbCmAYhROweMJWInj2dh_A` (UUID v3 from ANDROID_ID) |
| APK sha256 / cert / size | `/VEKrB9WsYGAMdFIVQPMo396qeQlYvT2bxZtu1gMPgw=` / `OKD31QX+GP7GT780Psqq8xDb15k=` / 127447992 |
| did | `TP1A.220624.014.A325FXXSBDXJ1` |
| device_ram | `3,56` (comma — ru locale) |
| `<success>` | `t=1788392932 location=vll pn=6615@s.whatsapp.net lid=4312@lid abprops=185 creation=1788392926` |

## Open questions (honestly not established by the reverse)

- part of the `X/C1KE`/`A03` constructor does not decompile in jadx's normal
  mode (the logic is presented from a fallback dump).

## How to read

1. [`docs/01-flow-overview.md`](docs/01-flow-overview.md) — the map of the
   whole flow and its dependencies;
2. [`docs/README.md`](docs/README.md) — the table of contents by stage;
3. then any stage folder; in each one every parameter is broken down from
   "where it sits on the wire" to "which native function produces it".

---

## Author & Contact

All reverse engineering, analysis and documentation were produced by the: **[https://t.me/Premium_SMS_Messenger](https://t.me/Premium_SMS_Messenger)**

Feel free to reach out with any questions, as well as to obtain a ready-made
library implementation built on the basis of this reverse engineering (Go).

**If this work was useful to you, you can thank me with the help of USDT (TRC-20).:**

```
THSLZyomC5h7kmQSNPqBSCR6eD1YTD1RXr
```
