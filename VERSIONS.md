# Versions covered by this reverse engineering

| Version | versionCode | Status | Coverage |
|---|---|---|---|
| **2.26.35.75** | 263507522 | base, full reverse | all documents, live registration and first login captures |
| 2.26.37.73 | 263707322 | spot verification | selected facts (see references in the docs) |
| 2.26.40.76 | 264007600 | native dumps | `libwhatsapp.so`, `libwhatsappmerged.so` |
| **2.26.44.90** | 264409011 | re-verified (Nov 2026) | login handshake, ClientPayload, post-login, native libs |

## What applies to 2.26.44.90

- Full delta: [docs/13-delta-2.26.44.90.md](docs/13-delta-2.26.44.90.md)
- Live post-login capture: [docs/14-post-login-capture-2.26.44.90.md](docs/14-post-login-capture-2.26.44.90.md)
- Attestation: [docs/12-post-login/keystore-attestation-2.26.44.90.md](docs/12-post-login/keystore-attestation-2.26.44.90.md)

The mainline documents describe 2.26.35.75. Where a fact changed, the document carries a
"2.26.44.90" note with a link to the delta.

## Verification artifacts

| Item | Path |
|---|---|
| APK 2.26.44.90 | `apk_update/pers_2.26.44.90_base.apk` |
| jadx tree | `jadx_264409011/sources` (83 889 classes) |
| smali | `smali1` (classes.dex), `smali2` (classes2.dex) |
| Native libraries | `natives_new/decompressed/` + `natives_new/report_264409011.json` |
| Frida agent | `hooks/agent/agent.ts` (compiled with `frida-java-bridge`) |
| Live capture | `hooks/wa_agent_log_registration.txt` |
