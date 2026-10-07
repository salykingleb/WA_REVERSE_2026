# Integrity blob encryption key — the shared AES-256-CBC envelope for gpia / _gi / _gg / bi / integrity_payload / jws

## Place in the flow

All encrypted integrity parameters of the registration HTTP requests `/v2/code` and `/v2/register`
(gpia, _gi, _gg) and the nested bi blob inside _ga, as well as the post-registration XMPP-login blobs
(integrity_payload — the response to the server `<safetynet><integrity nonce='…'>`, the `<jws>` envelope — the response
to `<gpia><request nonce='…'>`) are encrypted with **the same envelope and the same key**,
derived from the authkey of the current registration session:

1. The Java layer `X/IF6.java:553-567 (A0S)` calls `JniBridge.jvidispatchIOO(7, app, wajCtx)`
   (native state initialization) and `JniBridge.jvidispatchOOO(16, app, wajCtx)` — the native
   code returns a `Map<String, ByteArray>` of underscore parameters (_gi, _gg, _ga, _ge, _gp, _gs),
   which is entirely `putAll`-ed into the request parameters (IF6.java:694, 804, 1417, 1427, 1566, 1576, 1694, 1698).
2. gpia arrives via a separate path: `X/IF6.java:594-614 (A0V)` with AB flag 3753 enabled → the coroutine
   `J6X case 12` (Play Integrity) → `C36467FyF case 2` → `JniBridge.jvidispatchIIDOOOO(errorCode,
   shaRetryDelay, token, app, callback DSY, wajCtx)` — native builds the gpia JSON, encrypts it and returns
   the string via the callback → `map.put("gpia", B0W.A1Z(...))`.
3. Encryption is performed **entirely natively** (libwhatsapp.so); what goes on the wire are base64 strings,
   which Java additionally percent-encodes (`X/HSJ.A00`, RFC3986, `%XX` UPPERCASE).

Important: _ga as a wire parameter is an OPEN JSON, only the nested bi is encrypted; _ge and _gp are
open values. The envelope covers: gpia, _gi, _gg (fully), bi (inside _ga), integrity_payload
and jws (XMPP login).

## Wire format (+ live example)

The format of each encrypted blob's value:

```
base64.Std( IV[16 random bytes] || AES-256-CBC ciphertext(PKCS7(JSON string)) )
```

- base64 is STANDARD (alphabet `A-Za-z0-9+/`, padding `=`).
- After base64, Java wraps the value in percent-encoding: `+`→`%2B`, `/`→`%2F`, `=`→`%3D`.
- Live example (bi from _ga, captured 2026-09-22, session 1):

```
bi = Bge8ej5JWBp5Fi0v9KnANajtzs0+L45Q8FyEswRJUhzNsgKZyyweGl5KFNk//MVHQSxziPIlHMAHtfI+ST1LBA==
```

88 base64 characters (`==` at the end) = 64 bytes = IV 16 B || ct 48 B; the plaintext is a 44-character
base64 string of 32 bytes, PKCS7 padding 4×0x04 (44 → 48). In the open _ga JSON both `/` characters of the bi blob
are escaped as `\/` (`KFNk\/\/MVHQ`), `+` and `=` are not escaped.

## How it is formed in the application

Key derivation (confirmed on two independent live sessions):

```
authkey (b64url on the wire, 43 chars) → raw 32 bytes
raw 32 B → STANDARD base64 (44 characters, padded with "=")
AES key (32 B) = SHA256( ASCII bytes of this 44-character std-base64 string )
```

That is, what is hashed is NOT the authkey itself and NOT its raw bytes, but the **textual standard-base64 representation**
of the authkey. Live examples of both sessions:

| Session | WA version | authkey (b64url) | AES key = SHA256(std-base64(authkey)) |
|---|---|---|---|
| 1 (2026-09-22, /v2/code + /v2/register) | 2.26.35.75 | `rsgyL8y0zyOXIJT4KofOJGJaxrlKUp6TuBsPvGzZAzs` (std-b64: `rsgyL8y0zyOXIJT4KofOJGJaxrlKUp6TuBsPvGzZAzs=`) | `R41B51Od02uXyjkarGGxbrDvQghVHzMkl9W+HOQDJK4=` |
| 2 (2026-09-28, registration + XMPP login) | 2.26.37.73 | `Me_A3cdy1KTFeg8dfS-f27vIVDFeRMqXhOJMifg-b2w` | `pjhfD73u7ZYPErnsKS6NCcqYU/rzGIYKsh5vE0Uw5qo=` |

Session 1: with this key, 7 blobs were cleanly decrypted (gpia/_gi/_gg in code and register + bi), PKCS7
is valid for all of them. Session 2: 5 blobs of a single installation (gpia, _gi, _gg, integrity_payload, jws).
The formula was confirmed by the second independent session — the key exists one per authkey, all blobs
of the installation are decrypted with it.

## Native implementation (libwhatsapp.so)

Functions (a runtime dump of the loaded unpacked libwhatsapp.so from
`/data/data/com.whatsapp/files/decompressed/libs.spo/`, load base 0x7cfd633000):

- **`_WAJIntegrityCreateEncryptedAES256CBC` @ 0x7cfdd3b3a4** (rx+0x5b3a4) — the envelope itself:
  1. `0x7cfdd7d5fc(0x10)` — a buffer of **16 bytes = IV**;
  2. the EVP chain `CipherCreate → CipherCreateUpdate → CipherCreateFinal` (error strings in rodata:
     `"failed to encrypt message: CipherCreate"`, `"...CipherCreateUpdate"`, `"...CipherCreateFinal"`)
     — AES-256-CBC with padding (Final = PKCS#7);
  3. the output = the concatenation of **IV || Update-out || Final-out** (memcpy in this order, total length
     = len(IV)+len(ct1)+len(ct2));
  4. the result is base64-encoded (std, with `=`).
- **`WAJIntegrityCreateEncryptedStringUsingKey`** (call @0x7cfdd3b828-0x7cfdd3b8a0) — the key-selection
  wrapper: the key is taken from the storage **by name**: `[ctx+0x50] → … → key name, fallback
  "default"` (0x7cfdd3b338). The SHA256(std-base64(authkey)) derivation is performed inside this chain
  (not traced natively to the end, but externally validated by the byte-for-byte decryption of all live
  blobs of both sessions — no discrepancies).
- The same envelope is called from the _ga builder for bi: `bi ← 0x7cfdd3b3a4 (the x2 argument of the builder
  @0x7cfdd30864)`.

## Value selection conditions

- The key is hard-bound to the session's authkey: the authkey changed — the key changed, old blobs cannot be read
  with the new key (and vice versa).
- The IV is 16 random bytes per act of encryption; at the same time, within a single registration the _gi and
  _ga (bi) blobs in /v2/code and /v2/register are **byte-for-byte identical** (the value is reused/cached
  between the two requests), while gpia and _gg are recreated (fresh IV + fresh Play Integrity token).
- The escaping `/`→`\/` is performed ON the serialized JSON, AFTER serialization and BEFORE encryption —
  this is exactly how the live plaintext is reproduced (all `/` inside string values as `\/`: paths,
  base64 values of sha256/_dh/_is/_p; `+` and `=` are NOT escaped).
- **The bi nuance**: in integrity_payload (XMPP login) the nested `_ga.bi` is inserted into the JSON **AFTER**
  escaping — the slashes inside bi remain bare (`"bi":"kjqKmWIZ...Hnf/l8HK...b/Jys..."`), whereas
  `_dh`, `_is`, `aid` in the same blob are escaped. In the open _ga parameter (HTTP) it is the opposite:
  bi's slashes are escaped (`\/`) together with the whole JSON.

## Cryptography and encoding

The final chain for gpia/_gi/_gg/integrity_payload/jws:

```
1. Assemble the JSON (protobuf-JSON style, compact, no spaces)
2. Escape all "/" → "\/" (except the nested bi in integrity_payload)
3. AES-256-CBC: key = SHA256(std-base64(authkey)), IV = 16 random bytes, PKCS7
   (padding is ALWAYS added, even when the length is a multiple of the block size)
4. out = IV || ciphertext
5. base64.StdEncoding (with "=")
6. (HTTP) percent-encoding of the values by the Java side: X/HSJ.A00 → %XX UPPERCASE
```

Reproduction check: reassembling the live _ga value via the chain
(b64 → replace(`/`→`\/`) → JSON → percent-escape) yields a string BYTE-FOR-BYTE equal to the intercepted one.

## Examples

- Session 1 key: `R41B51Od02uXyjkarGGxbrDvQghVHzMkl9W+HOQDJK4=` — decrypts
  gpia[code], gpia[register], _gi (both requests, the blobs are equal), _gg[code], _gg[register], bi.
- Session 2 key: `pjhfD73u7ZYPErnsKS6NCcqYU/rzGIYKsh5vE0Uw5qo=` — decrypts gpia, _gi, _gg
  of the 2.26.37.73 registration and the login blobs: integrity_payload (768 bytes = 48 AES blocks) and the jws envelope.
- Structure of the bi blob: plaintext `"nZSX05b23Fax2H/ZCTptouz3dcvpv/ifppGSfUfqXnM="` (44 chars, 32 bytes)
  → PKCS7 to 48 bytes → 3 AES blocks → +IV = 64 bytes → base64 88 chars with `==`.
