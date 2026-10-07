# Signal keys (Signal key bundle)

**Stage A** · the `04-signal-keys/` folder of the WhatsApp 2.26.35.75
(263507522) reverse-engineering documentation.

The Signal key system registered together with the number: authkey,
e_keytype, e_ident, e_regid, e_skey_id, e_skey_val, e_skey_sig —
generation, storage, formats.

## Documents in this folder

| Document | Title |
|---|---|
| [authkey.md](authkey.md) | authkey — public key of the client's static X25519 keypair (ClientStaticKeyPair) |
| [e_ident.md](e_ident.md) | e_ident — Signal public identity key (X25519, 32 bytes without prefix) |
| [e_keytype.md](e_keytype.md) | e_keytype — elliptic curve type of the key bundle (constant 0x05) |
| [e_regid.md](e_regid.md) | e_regid — Signal protocol registration id (4 bytes big-endian, 1..2147483646) |
| [e_skey_id.md](e_skey_id.md) | e_skey_id — signed prekey identifier (3 bytes big-endian, 1..16777214) |
| [e_skey_sig.md](e_skey_sig.md) | e_skey_sig — signed prekey signature with the identity key (curve25519sha512, 64 bytes) |
| [e_skey_val.md](e_skey_val.md) | e_skey_val — signed prekey public key (X25519, 32 bytes without prefix) |
| [key-system-overview.md](key-system-overview.md) | Generation and storage of the key system at first registration (summary) |

Navigation for the whole guide: [docs/README.md](../README.md). Start: [Flow Overview](../01-flow-overview.md).
