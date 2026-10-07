# ACS `privatestats`: blind signatures version 1 and version 2

WhatsApp 2.26.35.75 (263507522). Wire: live capture of the first login. ACS (Anonymous Credential Service) — two exchanges per session: version 1 immediately after `<success>` (02:48:52.729, assembled by the native code) and version 2 `WA_StatusMusic` later (02:49:40.800, assembled by Java). Both are `<iq xmlns='privatestats' type='get'>` with a `<sign_credential>` node.

## Version 1 — immediately after success

### Request (native)

The IQ is assembled by the native code; Java only sends it — `JniBridge` case 7 requires exactly 32 bytes of blinded value:

```java
C0P0.A04(bArr2, 32L, 32L);          // ровно 32 байта
c05630Oz5.A01 = bArr2;              // blinded_credential
c0P1("version", "1")
// iq xmlns=privatestats type=get
```

The live request of this session (02:48:52.729, id `09`):

```xml
<iq xmlns='privatestats' type='get' to='s.whatsapp.net' id='09'>
 <sign_credential version='1'>
  <blinded_credential>0Gku3EuoyKjC5wNKuV7bUgx8Mhh/bciYI+hSNqu+1Go=</blinded_credential>
 </sign_credential></iq>
```

44 base64 characters = exactly 32 bytes. There is no project name in v1.

### Server response (02:48:53.726, after ~1 s)

```xml
<iq from='s.whatsapp.net' type='result' id='09'>
 <sign_credential t='1788392933'>
  <signed_credential>+HO1wabr7vwod+SVaUIrZ2RqQjfx7udqAe813xoignc=</signed_credential>
  <acs_public_key>pHJHuDwZRWpKuH7QQoroxIeGEeyjCzsLugK/PVfVlvA=</acs_public_key>
  <dleq_proof><c>UtzPkflJfXPRxGC+NX9wGlA304AEd99GsZLilVl7+Q4=</c>
              <s>sVv7+1KAWE+Z8Vau1pzsPuRUpG85uqNGGIrep/KdgQQ=</s></dleq_proof>
  <config_id></config_id>
 </sign_credential></iq>
```

Composition: signed_credential 32B + acs_public_key 32B + dleq_proof c,s of 32B each + config_id (empty here). An empty `config_id` means there was no child node with bytes and `null` went into the native code as the third argument — this is not an error.

### Processing: jvidispatchIOOOO id 4

The listener is `X/C35334FfE`. On success (`C5N`) it takes `signed_credential`, `acs_public_key` and the raw `config_id`, then carries everything into the native code:

```java
JniBridge.jvidispatchIOOOO(4, wajContext, bArr, bArr2, bArr3);
```

id 4 in the dispatcher (`0x7cfdd1dd00`, branch `0x7cfdd1df24`): the three `jbyteArray`s are unpacked by a single helper `0x7cfdd17fdc`, waj via `0x7cfdd17998`, and everything goes into the sink **`0x7cfdd22b84`** — the VOPRF core. The sink's body is 13 instructions and one call; the function sends nothing outward: the signature settles inside the WAJ context for further private metrics.

Errors: a delivery failure and an IQ-error call the same id 4 with three `null`s and write the `WamSignCredential` event with code 3. Success is code 1. The event's `A03` field on this path is always 1 — the version 1 marker.

## Version 2 — `WA_StatusMusic`

The live request (02:49:40.800, id `05d`):

```xml
<iq xmlns='privatestats' id='05d' type='get' to='s.whatsapp.net'>
 <sign_credential version='2'>
  <blinded_credential>FAoA8WLwvqaJIMGg1xdjHhbQPpPd+tNXhjsddE8r/m8=</blinded_credential>
  <project_name>WA_StatusMusic</project_name>
 </sign_credential></iq>
```

The response at 02:49:41.464 — the same signature/key/DLEQ triple, `project_name` as an echo, `config_id` empty again.

This one is already Java, `X/C22310zA.A00`: it sets `version='2'`, `project_name`, an optional `config_id`. The token is prepared by `X/C22330zC.A03`:

- the token length from the prefs `token_length`;
- the blinding factor is retried up to 256 attempts, the mask `bArr2[31] &= 31`, the result is compared against the group order `NativeVOPRFExtension.L`;
- the `blind(...)` call; for projects from the `X/C22200yw.A0A` list — the `VoprfEd25519` variant;
- the original token and the factor are written to prefs (`next_original_token_string`, `blinding_factor_string`) so the signature can be verified when it comes back.

## WamSignCredential error codes

| Code | When | Behavior |
|---|---|---|
| 1 | success (`C5N`) | the signature goes into id 4 |
| 3 | delivery failure / IQ code 500 | request retry |
| 5 | the blinding factor did not converge within 256 attempts (`cannot generate valid blindingFactor`) | abort |
| 7 | `blind(...)` returned null | abort |
| 10 | the response has no signature or key | stop |
| 11 | other error | stop |
| — | the response arrived with a foreign IQ id | full reset of `C22310zA.A02` |

For v1, codes 1 and 3 apply; the event's `A03` field is always 1.

## Why this is on the login wire

- v1 is sent by the native code 4 ms before the server's gpia nonce (.729 versus .736) — the client does not wait for instructions: the blind token is needed by the WAJ context itself for private statistics (`w:stats` with encrypted counters goes out at .700 and then continuously, e.g. id 067 at 02:49:41.997).
- The DLEQ proof (c, s) lets the client verify, without revealing the token, that the signature was computed with that very `acs_public_key` and over that very blinded value — the VOPRF protocol.
- `t='1788392933'` in the v1 response = the login second + 1 — the signature is bound to the moment of the session.

## Where to look

- v1 sending: `com/whatsapp/wamsys/JniBridge.java` (case 7, around line 1300), listener `X/C35334FfE.java`.
- Processing: dispatcher `0x7cfdd1dd00`, id 4 branch `0x7cfdd1df24`, sink `0x7cfdd22b84`, unpackers `0x7cfdd17fdc` / `0x7cfdd17998`.
- v2: `X/C22310zA.java` (`A00`, `C5N`, the `A02` reset), `X/C22330zC.java` `A03`, prefs `token_length`, `next_original_token_string`, `blinding_factor_string`; `X/C22200yw.A0A` (the VoprfEd25519 project list); `NativeVOPRFExtension.L` (the group order).
- Live wire: 02:48:52.729 / 02:48:53.726 (v1), 02:49:40.800 / 02:49:41.464 (v2).
