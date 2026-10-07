# _lh — hash of the native library from nativeLibraryDir (a new field in 2.26.37.73, the login integrity_payload)

## Place in the flow

`_lh` is a NEW field: in the _gi parameter of version 2.26.35.75 it is ABSENT (it is not in the audit of the live 35.75
capture), and in 2.26.37.73 (versionCode 263707322) it appeared in **integrity_payload** — the blob that the
client sends immediately after the first XMPP login following registration, in response to the server stanza
`<safetynet><integrity nonce='ATbE74Q…'>` (the native call `jvidispatchIOOOO` **id=5** in the Frida log).

The exact key order of the 2.26.37.73 integrity_payload (a live login dump of 2026-09-28):

```
aid, _dh, _lh, _iln, _ln, _ge, _gp, _ga, _ip, _isb, _n, _icr, _is
```

— `_lh` is third, right after `_dh`. The blob as a whole is encrypted with the shared envelope (AES-256-CBC,
key = SHA256(std-base64(authkey)), see encryption-key.md); the live blob's
size is 768 bytes (48 AES blocks). The September blob of 2.26.35.75 had the same size of 768 bytes — apparently
the same structure (including `_lh` and `_n`), but there is no decryption of it.

The `_lh` name itself is present in libwhatsapp's obfuscated string table as early as 2.26.35.75
(recorded in the list of "additional table keys absent from the live _gi capture": `_das, hwfp, _lh, _n,
sv, sb, _acg, _acj`) — i.e. the field was laid into the native code in advance and enabled in version 37.73
in the login payload (in _gi it is still absent).

## Wire format (+ live example)

Inside the decrypted integrity_payload — the string field `_lh` with standard base64 (32 bytes
of the SHA-256 digest, padding `=`):

```json
{"aid":"VM60VU78\/F6NzE1CSRYipwztp4PCM\/2hlFoIdUZBb4g=",
 "_dh":"jjyxvURV8j4F8rILRZttTITKv0JCvJG6F52K32iZ\/Ag=",
 "_lh":"oGLRVumypzq8fegSIB+MKwj1iNrmarhbZYDdj8yLCns=",
 "_iln":"i7ls4vAnG2Bvmsp8NDKh7uREGcY4q8Hi4JtdCLix+Cc=", ...}
```

Live value (SM-A325F, the first login after registration on 2.26.37.73):

```
_lh = oGLRVumypzq8fegSIB+MKwj1iNrmarhbZYDdj8yLCns=   (the digest's hex starts with a062d156…)
```

(inside the blob all `/` of the other fields are escaped as `\/`; the live `_lh` value has no slashes.)

## How it is formed in the application

The formula (the final reverse of 37.73; the field's existence re-verified against raw data — the substring
`,"_lh":"oGLRVumy…",` is present in the decryption, the JSON parses):

```
_lh = Base64( SHA-256( the file <ApplicationInfo.nativeLibraryDir>/<name>.so ) )
```

Nomenclature: `_lh` — "**l**ibrary **h**ash". The key difference from the neighbors: `_ln`/`_iln` hash
the **names** of libraries, `_lh` — the **content** of one .so file. The field belongs to the family of 32-byte
SHA-256 digests in base64, whose neighboring formulas are proven:

| Family field | Formula |
|---|---|
| `_is` | SHA-256(base.apk) |
| `_icr` | signing certificate digest |
| `_dh` | SHA-256(classes.dex) |
| `_iln` | b64(SHA-256(concat(sorted(names of `*.so` from dataDir)))) |
| `_ln` | b64(SHA-256(concat(sorted(names from nativeLibraryDir)))), the fallback constant b64(SHA-256("signal_error")) |
| `_lh` | b64(SHA-256(the content of one `.so` from nativeLibraryDir)) — the only field hashing a library's content |

Step by step along the 2.26.37.73 native chain (offsets in that build's libwhatsapp):

1. **The directory**: the ApplicationInfo JNI field by sid **0x2c = "nativeLibraryDir"** of the obfuscated
   string table (the AES-128-ECB decryption was confirmed back in the 35.75 reverse: sid 0x2b="dataDir",
   0x2c="nativeLibraryDir"; the table has 106 entries, "nativeLibraryDir" is entry 44 (0x2c), while the
   `_lh` key itself has sid **0x44**).
2. **The name**: the string is returned by a static Java method (the wrapper 0x6bb95c → the call 0x6c0ff8(1)), and the suffix
   **`.so`** is appended to it. The only remaining unknown is which exact string
   arrives from Java on argument 1 (the name's GOT strings are lazily encrypted). The candidate per the reverse is
   `libwhatsapp` (index 1), i.e. the final file `libwhatsapp.so`.
3. **Opening the file**: `MCFURLCreateWithCStringFileSystemPath` (the regular file system, NOT zip parsing).
4. **The hash**: `mbedtls_sha256_update` / `mbedtls_sha256_finish` over the raw fread bytes, **limit −1
   (the whole file)** — unlike shatr/_ist, where the limit is 0xA00000.
5. **Encoding**: `MCIBase64EncodeCreateString` → standard base64.

Error codes (result strings instead of the hash): an empty name from Java → `invalid_library_name`; the URL
was not created → `failed_url_creation`; the hash did not assemble → `failed_sha`.

## Native implementation (libwhatsapp.so 2.26.37.73)

- The "name/path/result" wrapper: offset **0x6da638** — takes the name Java method's result,
  builds the path, calls the hasher, returns the string.
- The name Java method: **0x6bb95c → 0x6c0ff8(1)** (static, JNI; its output + `".so"`).
- The hasher: **0x6CE9EC** — fopen/fread → `mbedtls_sha256_update/finish` (limit −1) → `MCIBase64EncodeString`.
- Opening: `MCFURLCreateWithCStringFileSystemPath` (file system, not ZIP).
- The string table: the `0x7cfdd76a64(sid)`-compatible scheme of 37.73; sid 0x2c = "nativeLibraryDir",
  sid 0x44 = "_lh" (entry 44 of 106 — the number coincidence is accidental: 0x2c=44 decimal — the directory,
  0x44=68 — the key).
- A reverse peculiarity: the hasher's code pages are lazily encrypted — an ad-hoc NativeFunction call
  crashes with SIGILL; hooks are placed only on live calls.

## Value selection conditions

- The value exists only when the file `<nativeLibraryDir>/<name>.so` was successfully opened; the live
  `oGLRVumy…` is a SUCCESSFUL hash (not an error string) ⇒ the file really existed in nativeLibraryDir at
  the moment of the first login on 28.09.2026 07:53.
- The dumped-device paradox: **right now there is no file in nativeLibraryDir** (the directory is empty — the build with
  extractNativeLibs=false delivers the libraries via the superpack to
  `/data/data/com.whatsapp/files/decompressed/libs.spo/`). Verified on the device (root, sha256sum):
  none of the 82 libs.spo modules, nor libs.so / libsuperpack.so / libunwindstack_binary.so from the
  arm64 split, nor libgojni.so from the proxyservice split yields the digest `a062d156…`. The libs.spo
  directory is irrelevant — the hasher reads exactly one file from nativeLibraryDir.
- An indirect constraint: the mtime of the libs.spo modules (27.09 09:31) is OLDER than the first login (28.09 07:53) —
  the files were not rewritten at the moment of login; hence, the sought .so is not a freshly created copy
  from the superpack (either it is created earlier and deleted, or it is a different file).
- The profile value for 2.26.35.75 (the candidate `libwhatsapp.so`, index 1 per the reverse):
  `_lh = b64(SHA256(libwhatsapp.so 2.26.35.75))` = **`euEs+17faTS6Mt7WHjzn7spB95xywNpuimy0sWbyuME=`**
  (the file is 13 951 456 bytes, recomputed from the gs_2.26.35.75 dump). The backup candidate (the name `merged`):
  `jzHg8a6z…`.
- The value is stable for the pair (device profile, file version); it changes when the .so version changes.

## Cryptography and encoding

- SHA-256 of the whole file (mbedtls, limit −1) → MCIBase64 → standard base64 (44 characters, `=`).
- The field lives only inside the ENCRYPTED integrity_payload (it is not sent out separately);
  the envelope is the shared AES-256-CBC (key SHA256(std-base64(authkey)), IV 16 B ‖ PKCS7, base64.Std).
- Escaping: `/` in the payload's values is escaped as `\/` BEFORE encryption (except the nested
  `_ga.bi`, inserted after escaping); the live `_lh` has no slashes.

## Examples

- Live (37.73, login): `oGLRVumypzq8fegSIB+MKwj1iNrmarhbZYDdj8yLCns=`.
- Candidate (35.75, libwhatsapp.so): `euEs+17faTS6Mt7WHjzn7spB95xywNpuimy0sWbyuME=`.
- Error values: `invalid_library_name` / `failed_url_creation` / `failed_sha` (strings instead of base64).

## Open questions

1. What string arrives from the Java method 0x6c0ff8 on argument 1 (the name without `.so`)? — the formula's only
   unknown; the name's GOT strings are lazily encrypted, the file is no longer on disk, the bytes from
   disk are not reproducible.
2. Where did the file in nativeLibraryDir come from at the first login and why is it gone now (unpacking
   on the first start with subsequent deletion? a separate split?).
