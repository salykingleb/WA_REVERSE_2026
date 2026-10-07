# _ge — environment flags (open JSON {"sb":false,"sv":false})

## Place in the flow

The `_ge` parameter is attached to all registration HTTP requests (`/v2/code`, `/v2/register`,
makeConsentRequest, pre-chatd AB check, verifySecurityCode, registerPhoneNumber, LinkedUsersActivity)
by the same native package as _ga/_gp/_gi/_gg:

```
X/IF6.java:553-567 (A0S):
    JniBridge.jvidispatchIOO(7, application, wajContext)     // native state initialization
    map2 = JniBridge.jvidispatchOOO(16, application, wajContext)
    map.putAll(map2)                                          // _ge among the underscore parameters
```

A0S call sites: `RequestCodeRepository$requestCode$2.java:402` (/v2/code), `IF6.java:694/804/1417/1566`
(/v2/register etc.). The `_ge` string is absent from the Java/DEX layer (verified by a scan of the string tables of all
classes*.dex — the scan method finds `gpia` and `authkey`, but not `_ga`/`_ge`/`_gp`), the value arrives
from native as a ready ASCII string. On the wire in /v2/code _ge is 12th, in /v2/register — 15th.
After login the same value is embedded as an object into the XMPP integrity_payload (the `_ge` field).

## Wire format (+ live example)

An open (unencrypted) compact JSON of exactly two boolean fields, the order `sb, sv`:

```json
{"sb":false,"sv":false}
```

After QueryEscape (Java X/HSJ.A00 — RFC3986, %XX UPPERCASE; equivalent to url.QueryEscape):

```
%7B%22sb%22%3Afalse%2C%22sv%22%3Afalse%7D
```

(parsing: `%7B%22sb%22%3Afalse` = `{"sb":false`, `%2C` = `,`, `%22sv%22%3Afalse%7D` = `"sv":false}`).

Live confirmation: both captures (/v2/code and /v2/register of 2026-09-22, 2.26.35.75) and the 2.26.37.73 login
integrity_payload contain the same `{"sb":false,"sv":false}` — the value is byte-for-byte identical
inside the session and between the _ge parameter and the payload's nested `_ge`.

## How it is formed in the application

- The value is generated entirely natively in the merged library inside the superpack container libs.so
  (dispatcher 16); it arrives in Java as a ready string.
- The field names `sb`/`sv` are in the AES-128-ECB-obfuscated string table of libwhatsapp
  (`0x7cfdd76a64`): the sid 100-103 cluster = `sv, sb, aid, did` (right after the _ga cluster sid 94-99
  ai/ap/ae/mu/mp/bi and sid 93 `/proc/sys/kernel/random/boot_id`). They are absent from the open string table of the
  merged library (stream_000 of the unpacked superpack) — `\x00sb\x00`/`\x00sv\x00`
  in streams 000/009 were recognized as random coincidences in glif-name tables and data.
- The flag semantics was decoded byte-for-byte: the native `_ge` builder @ **0x7cfdd30b08** in
  libwhatsapp.so. It is the mini-check "is the device in a VM?" — exactly the same first two checks
  as in the libwasafe stat-path table for `_gs`:

  | Flag | Check | What it means |
  |---|---|---|
  | `sv` | `stat("/sys/module/vmw_pvscsi") == 0` | the VMware paravirtual SCSI driver is loaded → the device is under VMware |
  | `sb` | `stat("/sys/module/vboxsf") == 0` | `vboxsf` (VirtualBox shared folders) is loaded → the device is under VirtualBox |
  | `su` | OR over 9 su-paths | root detection; does NOT get into the request — it is inserted only under AB gate 14, which is default=0 (in the live capture of 2026-09-22 it was disabled even on the rooted SM-A325F) |

  On a normal phone both modules are absent → `stat != 0` → `false`; the insertion order is
  reversed (sv, sb), so on the wire it is always `{"sb":false,"sv":false}`.

## Native implementation (libwhatsapp.so / merged lib superpack)

- The `_ge` JSON builder — **0x7cfdd30b08** (libwhatsapp.so, the reverse in out3/E_native_map.md §2):
  performs `stat()` on the two virtualization module paths (`/sys/module/vmw_pvscsi`,
  `/sys/module/vboxsf`), with AB gate 14 enabled additionally an OR over 9 su-paths
  (the `su` flag), assembles the compact JSON and hands it to the parameter map.
- The issuing point is the same underscore-parameter map assembler @0x7cfdd3ffd4 (the dispatcher
  `jvidispatchOOO(16, app, wajCtx)`), from which the values get into the `Map<String, ByteArray>` map
  and are merged into the parameters by the Java side.
- The field names are extracted by the string decryptor `0x7cfdd76a64(sid)` (AES-128-ECB, the key = the decoy string
  + NUL, zero-padded to 16 bytes; the decoys lie in the open in rodata 0x7cfd9278xx-0x7cfd927cxx,
  4 pointer tables @0x7cfd9282b8/608/7b0/b00).
- The names of the `_ge` parameter itself and of the fields in the merged library's table are encrypted by a separate scheme
  (single-/multi-byte XOR and Caesar were brute-forced — not it; in place of the names are short binary blobs,
  e.g. the 21-byte `2e1e26113f472a531a4f0c1543082204324b576f6b` @stream_000+0x5b41d4, between
  `event_name` and `sent`). Cracking it requires an IDA analysis of the merged library's string decryptor.
- The encoding side is Java: the byte[] values are percent-encoded by `X/HSJ.A00` (called from
  `C40787I4g.A06`, RegistrationRequestBuilder, X/C40787I4g.java:74-83); the keys are added to the set
  `C40787I4g.A01` — the marker "the value is already encoded, do not encode again".

## Value selection conditions

- In all live dumps: `sb=false`, `sv=false` (both sessions, code and register, plus integrity_payload).
- The value is NOT recreated between /v2/code and /v2/register — the parameter is in the "identical" list
  (stable within an installation, like aid/_gp).
- The conditions for true are established exactly: `sv=true` — only under VMware (the `vmw_pvscsi` module loaded),
  `sb=true` — only under VirtualBox (`vboxsf` loaded). The third computed flag `su`
  (root detection, OR over 9 su-paths) does not get onto the wire: the insertion is blocked by AB gate 14
  with the default value 0 — in the live capture it is disabled even on a rooted device.
  `su:true` on the wire is a sign of forgery, it does not occur in real traffic.
- The server-side interpretation is unknown (an open JSON without value obfuscation).

## Cryptography and encoding

- No cryptography at all: the open ASCII JSON `{"sb":false,"sv":false}`.
- The wire encoding: RFC3986 percent-encoding with uppercase hex (`X/HSJ.A00` on the phone;
  equivalent to url.QueryEscape — the `~`/`*`/space differences never occur in this charset).
- Inside integrity_payload the value is embedded as a nested object `"_ge":{"sb":false,"sv":false}`
  without percent-encoding (there it is inside the encrypted blob).

## Examples

- Live /v2/code 2026-09-22 (a wire fragment): `...&_ge=%7B%22sb%22%3Afalse%2C%22sv%22%3Afalse%7D&...`
- Live integrity_payload 2.26.37.73 (a fragment of the decrypted plaintext):
  `"_ge":{"sb":false,"sv":false}` — the same value as in the HTTP parameter of the same installation.
