# /v2/register flow (OTP verification)

**Stage B** · the `09-register-flow/` folder of the WhatsApp 2.26.35.75 (263507522) reverse-engineering documentation.

The complete call map of code verification and the specific parameters: code, entered, has_play_store, passkey_login_status, vname, tos_version, 2FA, and additional parameters.

## Folder documents

| Document | Title |
|---|---|
| [call-map.md](call-map.md) | POST /v2/register (OTP verification flow) — complete map of calls and parameters |
| [code.md](code.md) | code — the 6-digit OTP code for number confirmation |
| [entered.md](entered.md) | entered — the numeric code of the OTP entry method |
| [extra-params.md](extra-params.md) | Additional conditional /v2/register parameters: fid, preloads_*, cred_token, old_phone_number, context, security_code (2FA) |
| [has_play_store.md](has_play_store.md) | has_play_store — indicator of an installed Google Play Store |
| [passkey_login_status.md](passkey_login_status.md) | passkey_login_status — stage of the passkey login attempt (wireToken) |
| [tos_version.md](tos_version.md) | tos_version — version of the accepted Terms of Service ("5") |
| [vname.md](vname.md) | vname — VerifiedNameCertificate (verified business name certificate) |

Navigation for the whole guide: [docs/README.md](../README.md). Start: [Flow overview](../01-flow-overview.md).
