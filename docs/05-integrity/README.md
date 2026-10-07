# Encrypted integrity parameters

**Stage A** · the `05-integrity/` folder of the WhatsApp 2.26.35.75 (263507522) reverse-engineering documentation.

AES-256-CBC blobs about installation integrity: gpia, _gi, _gg, _ga, _ge, _gp, _lh and the shared key of their encryption, derived from authkey.

## Documents in this folder

| Document | Title |
|---|---|
| [_ga.md](_ga.md) | _ga — installation ages block and anti-monkey flags (open JSON {bi, ap, ai, mp, ae, mu}) |
| [_ge.md](_ge.md) | _ge — environment flags (open JSON {"sb":false,"sv":false}) |
| [_gg.md](_gg.md) | _gg — encrypted Google Play Integrity result block (JSON {_ic, _it}: code + token) |
| [_gi.md](_gi.md) | _gi — encrypted installation attestation block (JSON of 10 fields: dex/lib hashes, firmware, APK) |
| [_gp.md](_gp.md) | _gp — hash of the manifest permission set (SHA256 of the sorted concatenation of 85 permissions) |
| [_lh.md](_lh.md) | _lh — hash of the native library from nativeLibraryDir (new field in 2.26.37.73, the login integrity_payload) |
| [encryption-key.md](encryption-key.md) | Integrity blob encryption key — the shared AES-256-CBC envelope for gpia / _gi / _gg / bi / integrity_payload / jws |
| [gpia.md](gpia.md) | gpia — encrypted Google Play Integrity APK attestation block (JSON of 8 fields about the APK and the token) |

Navigation for the whole guide: [docs/README.md](../README.md). Start: [Flow overview](../01-flow-overview.md).
