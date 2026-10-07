# After `<success>`: attestations and initial synchronization

**Stage D** · the `12-post-login/` folder of the WhatsApp 2.26.35.75 (263507522) reverse-engineering documentation.

The first second after login: prekey upload, keystore attestation, integrity_payload, Play Integrity jws, ACS privatestats, and initial account synchronization.

## Documents of the folder

| Document | Title |
|---|---|
| [acs-privatestats.md](acs-privatestats.md) | ACS `privatestats`: blind signatures version 1 and version 2 |
| [gpia-jws.md](gpia-jws.md) | Play Integrity after login: `<gpia><request nonce>` → `<ib><gpia><jws>` |
| [initial-sync.md](initial-sync.md) | Initial synchronization after success: all the remaining nodes |
| [integrity-payload.md](integrity-payload.md) | SafetyNet envelope: `<ib><integrity_payload>` (768 bytes) |
| [keystore-attestation.md](keystore-attestation.md) | Keystore attestation: `<ib><keystore_attestation>` |
| [prekeys-encrypt.md](prekeys-encrypt.md) | Prekey upload: `<iq xmlns='encrypt' type='set'>` immediately after success |
| [timeline.md](timeline.md) | Timeline of the first second after `<success>` |

Navigation for the whole guide: [docs/README.md](../README.md). Start: [Flow Overview](../01-flow-overview.md).
