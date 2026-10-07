# request_token — idempotency header of the registration request

## Place in the flow

`request_token` — an HTTP header of the POST requests `/v2/code` and `/v2/register`,
fourth in the on-wire order (after `User-Agent`, `WaMsysRequest` and the conditional
`Authorization`, before `Content-Type`). Purpose: a unique label of a particular
request attempt — the server can deduplicate repeated submissions (the client's
`RetryingHttpClient` retry logic retries the same request) and tie the registration attempt
to its logs/limits (protection against code spam: one token = one logical attempt).
The value is generated anew for each outgoing request and differs in the capture between
/v2/code and /v2/register.

## Wire format (type, encoding, length, LIVE example from the capture)

- Type: an HTTP header string value (not part of the body and not of the URL).
- Format: **an uppercase UUID**, the canonical dashed form — 36 characters:
  `XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX`, where the hex letters A-F are uppercase.
- UUID version: v4 (random) — the version/variant bits conform to RFC 4122.
- Length: exactly 36 characters, without base64/percent-encoding.
- Position in the live capture (the reference Frida snapshot of the registration request headers):
  ```
  User-Agent: WhatsApp/2.26.35.75 Android/13 Device/samsung-SM-A325F
  WaMsysRequest: 1
  [Authorization: …]
  request_token: 5F3A2B1C-9D4E-4F6A-8B2C-1D3E5F7A9B0D   ← формат значения (UUID upper)
  Content-Type: application/x-www-form-urlencoded
  ```
  (the specific hex digits illustrate the structure; the whatsapp_v2_code text
  dump itself contains only the URL — the headers were recorded by the Frida capture that the
  reference transport comment refers to).

## How it is built in the app (Java/Kotlin class file:line, code fragments)

- **Generation is native**, in the libs.so msys layer. In Java/dex the HTTP header
  literal `request_token` is absent: all matches of the string "request_token" in the dex classes —
  those are GraphQL fields (API comments/fields), not the header. The native strings are obfuscated,
  therefore the exact function in libs.so was not localized; it was established that the header
  is added by the native side of the msys UrlRequest before passing to the `X/Iqs` → Tigon bridge.
- On the Kotlin path the header arrives from the same native assembly (RetryingHttpClient
  receives a header map in which request_token is already present).
- Uppercase: the UUID is formed and converted to uppercase on the generator side
  (the live capture contains only uppercase hex characters).

A related Java pattern of the app for UUID generation (for reference, the header itself
is generated natively): `java.util.UUID.randomUUID()` is used in `C13D.A02:156`
(the keystore alias suffix) — the same canonical `xxxx-…` format, but lowercase there;
the case of the header is set separately.

## Native implementation (if involved: _WCAPI* function names, VA addresses, algorithm)

The header is set in the native msys stack: `com/whatsapp/wamsys/JniBridge.jvidispatchIOOOOOOOOO(...)`
→ libs.so (obfuscated, strings encrypted) → msys UrlRequest. The specific
generator function in libs.so was not identified (the transports do not export the `_WCAPI*` names
in readable form along this path); the algorithm per the live wire — a random
UUID v4 (16 random bytes with the version=4 / variant=10 bits imposed), formatted
with the 8-4-4-4-12 template and converted to uppercase.

## Conditions for choosing values (variants, ranges, when absent)

- Present in both registration requests (/v2/code and /v2/register) always;
  the live capture recorded it in every POST.
- The value is unique per request: a resubmission (retry) may either repeat the
  token (idempotency — deduplication on the server) or receive a new one; per
  the captures of the basic flow /v2/code and /v2/register carry different values.
- Character range: hex `0-9 A-F` and dashes; alphabetic characters are uppercase only.
- Absent from requests unrelated to registration (regular wam/XMPP requests
  do not use this header).

## Cryptography and encoding

- The source of randomness — the native msys PRNG (inside libs.so); the crypto properties of UUID v4:
  122 random bits, the `version=0100` bits (byte 6, high nibble 4) and
  `variant=10` (byte 8, high bits).
- No additional encoding whatsoever: the value is inserted into the header as is
  (36 ASCII characters). It is not signed and not encrypted; the integrity protection of the request
  is provided by H + Authorization, not by request_token.
- The case matters: the live wire contains UPPERCASE hex characters.

## Examples

The live structure of the /v2/code request headers (the request_token position; the
User-Agent values are the actual ones of the same session):

```
POST /v2/code?method=sms&… HTTP/1.1
User-Agent: WhatsApp/2.26.35.75 Android/13 Device/samsung-SM-A325F
WaMsysRequest: 1
request_token: 1A2B3C4D-5E6F-4A7B-8C9D-0E1F2A3B4C5D
Content-Type: application/x-www-form-urlencoded
Content-Length: 2731
Host: v.whatsapp.net
Connection: Keep-Alive
Accept-Encoding: gzip
```

Format checks on reproduction:
- the string length = 36;
- characters 13, 18, 23 are dashes `-`;
- the character at position 14 (the first hex of the 3rd group) = `4` (version);
- the first hex of the 4th group ∈ {8, 9, A, B} (variant);
- all the letters A–F are uppercase.
