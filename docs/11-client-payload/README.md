# ClientPayload login payload

**Stage C** · folder `11-client-payload/` of the WhatsApp 2.26.35.75 (263507522) reverse-engineering documentation.

The protobuf payload inside the handshake: all 48 fields — username, userAgent, connect fields, lc/oc, device classes, and the rest.

## Documents in this folder

| Document | Title |
|---|---|
| [connect-fields.md](connect-fields.md) | ClientPayload connection fields: 10, 12, 13, 15, 16, 43 |
| [device-classes.md](device-classes.md) | Fields 36 yearClass and 37 memClass — device performance classes |
| [lc-oc.md](lc-oc.md) | Field 24 lc (successful login counter) and field 23 oc (signature check) |
| [misc-fields.md](misc-fields.md) | Other ClientPayload fields: 41, 40, 45/46, 47, 18 |
| [pushname-sessionid.md](pushname-sessionid.md) | Fields 7 pushName and 9 sessionId |
| [structure.md](structure.md) | ClientPayload: how the encrypted login payload is assembled |
| [user-agent.md](user-agent.md) | Field 5 userAgent (X/C1Ht) — complete field map |
| [username-passive.md](username-passive.md) | Fields 1 username and 3 passive |

Navigation for the whole guide: [docs/README.md](../README.md). Start: [Flow overview](../01-flow-overview.md).
