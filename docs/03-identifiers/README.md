# Identifiers and Tokens

**Stage A** · the `03-identifiers/` folder of the WhatsApp 2.26.35.75 (263507522) reverse engineering documentation.

Unique installation and registration-session identifiers: id, backup_token, fdid, expid, aid, pid, advertising_id, access_session_id, token — plus the parameter builder that encodes them.

## Folder documents

| Document | Title |
|---|---|
| [access_session_id.md](access_session_id.md) | access_session_id — registration session UUID v4 → 16 bytes → base64 RawURL |
| [advertising_id.md](advertising_id.md) | advertising_id — Google Advertising ID (GAID), an optional parameter |
| [aid.md](aid.md) | aid — native device identifier: a SHA-256 chain from ANDROID_ID in base64 |
| [backup_token.md](backup_token.md) | backup_token — backup registration token, 20 bytes in SharedPreferences |
| [expid.md](expid.md) | expid — device ID for telemetry, UUID v3/v4 → 16 bytes → base64 RawURL |
| [fdid.md](fdid.md) | fdid — installation phone ID, a UUID v4 string |
| [id.md](id.md) | id — random identifier of a code request attempt, persistent per phone number |
| [pid.md](pid.md) | pid — OS process identifier (Process.myPid) |
| [request-builder.md](request-builder.md) | parameter_builder — how the /v2/code and /v2/register request parameters are assembled |
| [token.md](token.md) | token — code request anti-fraud token: HMAC-SHA1(secretKey, certDER || md5Hash || phone) |

Navigation for the whole guide: [docs/README.md](../README.md). Start: [Flow overview](../01-flow-overview.md).
