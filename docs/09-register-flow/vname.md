# vname — VerifiedNameCertificate (verified business name certificate)

## Position in the flow

A conditional POST /v2/register parameter: a "Verified Name" protobuf
certificate attached when registering/verifying a WhatsApp business profile.
In the consumer build of the client the first argument of the chain (bzp) is
always null, so the field is never sent and the app logs
`RegistrationHttpManager/verifyCode/vname-absent`. The presence of vname in
traffic is a marker of business registration (SMB binding).

## Format (+live example)

`vname = Base64(certificate.toByteArray(), 11)` — i.e. URL-safe Base64
without padding/line breaks (`Base64.URL_SAFE | NO_WRAP | NO_PADDING`,
flag 11 = RFC 4648 §5 "URL and Filename safe Alphabet" without padding),
over the serialized protobuf.

The certificate protobuf schema — X/BZP.java (VerifiedNameCertificate):

| Field | No. | Type |
|---|---|---|
| details | 1 | bytes (serialized Details) |
| signature | 2 | bytes (client signature) |
| serverSignature | 3 | bytes (server signature, on update) |

Details — X/C25975Bb8.java (VerifiedNameCertificate.Details):

| Field | No. | Type |
|---|---|---|
| serial | 1 | uint64 |
| issuer | 2 | string |
| verifiedName | 4 | string |
| localizedNames | 8 | repeated (BZO) |
| issueTime | 10 | uint64 |

Live example: in the 2026-09-22 consumer dump the field is absent
(vname-absent); a structure example (business variant):
`vname=<RawURL-Base64 marshal(
{details=marshal({serial=<uint64>, issuer="smb:wa", verifiedName="…"}),
signature=<64B curve25519>})>`.

## How it is formed in the app (class file:line, code)

```java
// X/IF6.java:367-373 (inside A0I)
if (bzp != null) {
    linkedHashMapA0O.put("vname", Base64.encode(bzp.toByteArray(), 11));
    str2 = "RegistrationHttpManager/verifyCode/vname-attached";
} else {
    str2 = "RegistrationHttpManager/verifyCode/vname-absent";
}
Log.i(str2);
```

The bzp path top to bottom:

```java
// VerifyPhoneNumber.java:1356 — the FIRST argument is null in the consumer build:
AbstractC49512Fp.A1X(new VerifyCodeUseCase$verifyCode$1(
        null /* = $verifiedNameCertificate = bzp */, h8d, h8a, code, method, ...));
// → VerifyCodeRepository$verify$2.$verifiedNameCertificate → IF6.A0I(bzp, ...)
```

Certificate generation is cut out of the consumer APK: the SMB branches are
replaced with `throw error("getVNameCertForVerifyTwoFactorAuth")` stubs
(VerifyTwoFactorAuth.java:405–409, WaConsentRepository.java:269–271) —
obtaining bzp != null from the consumer app is impossible.

## Value selection conditions

- The field is present ⇔ `bzp != null` ⇔ a business build (WhatsApp Business
  / SMB binding) attaches the verified name certificate during verification.
- Consumer app: the parameter is **never sent** (the live dump confirms
  this — vname is absent).
- Certificate field values (for the business variant, per historical external
  implementations and the schema):
  - serial — a random uint64 (generation is unavailable in the consumer
    build; the exact algorithm — ❓ Business APK only);
  - issuer — the issuer string; the business source — `"smb:wa"` (the
    literal is absent from the consumer APK — ❓);
  - verifiedName — the verified profile name (via business verification);
  - issueTime — the unix-time of issuance.

## Cryptography and encoding

- **Signature**: signature (field 2) — a signature with the client identity
  key (e_ident, curve25519) over the `marshal(details)` bytes; length 64
  bytes (r||s-style for 25519). The algorithm itself is absent from consumer
  Java (stub) — confirmed only by external implementations; historically the
  signature is a nacl-style sign over marshal(details).
- **Encoding**: `Base64.encode(..., 11)` = URL_SAFE|NO_WRAP|NO_PADDING — on
  the wire inside the map → bytes → HSJ.A00 percent-encoding (the URL-safe
  Base64 alphabet contains `-`/`_`, which the percent-encoder leaves as is).
- The protobuf schema and field numbers are confirmed by decompilation of
  BZP/C25975Bb8 (message-info strings `"\u0001\u0003…"` = 3 fields /
  `"\u0001\u0005…"` = 5 Details fields) and match the known def.proto
  VerifiedNameCertificate.

## Examples

- Consumer (live dump): the field is absent, the logs show `vname-absent`.
- Business registration (hypothetical wire):
  a `vname=dp8JYhIAEiM3c21iOndhGAEoADjrvbe0BQ==`-like RawURL-Base64 string
  (the length depends on verifiedName/localizedNames).
