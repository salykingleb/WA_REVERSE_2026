# Play Integrity after login: `<gpia><request nonce>` → `<ib><gpia><jws>`

WhatsApp 2.26.35.75 (263507522). Wire: live capture of the first login. The server sends at 02:48:52.736:

```xml
<ib from='s.whatsapp.net'><gpia><request
  nonce='AeGnsZKe2wR1JPjPnMSTdyTWCM0Kb-aPyn9WRhGFruxwfLyJBUEMkOIFzTYYIqGmMb6uZxsXFc5A'/>
</gpia></ib>
```

The nonce is 76 base64url characters without padding = **57 bytes**. The client responds with `<ib><gpia><jws>` at 02:48:57.266: a blob of 3968 base64 characters = **2976 bytes** of binary token (not a textual JWT: no `.` and no `eyJ`). The delay of **4.53 s** is the only login attestation step that waited for the external network (an asynchronous Google Play `Task`).

## The Java chain

On `<gpia><request nonce>`, `X/C27871Nj` calls `C1HI.A1J` (no separate log line); the latter posts message 254 — the nonce itself as `obj`:

```java
public static Message A0T(String str) {
    return Message.obtain(null, 0, 254, 0, str);
}
```

The handler is `X/C12G.BD5`, with a duration metric:

```java
((CYG) c27959COp.A01.A00.get()).A00("GPIA_DURATION");
JniBridge.jvidispatchIOOOO(3, obj, application, interfaceC31098Djv, jniBridge.getWajContext());
```

The object order for id 3: the nonce string, `Context` (application), the callback `Djv`, `wajContext`. The callback assembles the node and closes the metric:

```java
public void APR(String str) {
    ((CYG) ...).A01("GPIA_DURATION", "");
    byte[] bArrA1Z = B0W.A1Z(str);          // base64-декод токена
    // узел: <ib><gpia><jws>bArr</jws></gpia></ib>
    C0P0.A04(bArrA1Z, 1L, 9007199254740991L);  // длина >= 1
    c0oe.A0U(node, 371);
}
```

Java discards an empty token: the length must be at least 1.

## Native: id 3, `_WCAAPISendNonce`

In the dispatcher `Java_com_whatsapp_wamsys_JniBridge_jvidispatchIOOOO` (code `0x7cfdd1dd00`, slot `0x7cfe363a90`) the id 3 branch is `0x7cfdd1de70`. id 3 unpacks the arguments the same way as id 5 but calls `0x7cfdd3f2c0`, and that one calls **`_WCAAPISendNonce`**, stub `0x7cfdc78260`, body **`0x7cfdd3f360`** (122 instructions; name string in rodata: `0x7cfd79767b`). Inside is a direct JNI call to Android (`android/content/Context.getApplicationInfo`) and Play Core resolution via `FindClass`/`GetMethodID` (rodata from `0x7cfd8ee24f`):

```text
com/google/android/play/core/integrity/IntegrityManagerFactory
  create (Landroid/content/Context;)IntegrityManager
com/google/android/play/core/integrity/IntegrityTokenRequest
  builder ()Builder
  setCloudProjectNumber (J)Builder
  setNonce (Ljava/lang/String;)Builder
  build ()IntegrityTokenRequest
IntegrityManager.requestIntegrityToken
  (IntegrityTokenRequest;)Lcom/google/android/gms/tasks/Task;
IntegrityTokenResponse.token ()Ljava/lang/String;
```

This is the genuine (classic) Play Integrity API: the server nonce from `<request>` goes into `setNonce`; the cloud project number is the one the preparation (`jvidispatchIIIIDOOO`, `0x7cfdd21044` → `0x7cfdd31dd8`) wrote via `setCloudProjectNumber` — **62949436** (live confirmation on 2.26.37.73: `jvidispatchIIIIDOOO args=["0","62949436","855397460"]`, first argument 0 — AB flag 12964 is off). The `Task` completes asynchronously; `token()` returns as a string to the `APR` callback.

Errors of the same JNI are visible as rodata strings: `RequestIntegrityToken failed due
to null out parameter`, `Unexpected null result while handling
IntegrityTokenRequest`, `Unexpected error %d while handling
RequestIntegrityToken`. In this capture the callback received the string — the `<jws>` is non-empty.

## The live envelope (decryption from 2.26.37.73)

The same mechanism on the 2.26.37.73 capture (263707322): nonce `AeECDYl7Vg8SdviNdJC9tBUl-hvSD6qtO_ytKKnZelmwZVJ8tYKdMOzOO5XoGFDkaUiuEBCbJPCoxr0`, `<jws>` 3692 b64 chars ≈ **2768 bytes**. Decryption with the same envelope (AES-256-CBC, key `SHA256(std_base64(authkey))`):

```json
{"retries":2,"code":0,
 "packageName":"com.whatsapp",
 "sha256":"1oUpYgdN0ehe\/QGjX23NFK+oj8riGuiJQJIdhGCsJus=",
 "token":"eyJhbGciOiJBMjU2S1ciLCJlbmMiOiJBMjU2R0NNIn0.cRBoBfP5….IGM5reMakLSxk7x1CnkYfQ",
 "nonce":"AeECDYl7Vg8SdviNdJC9tBUl-hvSD6qtO_ytKKnZelmwZVJ8tYKdMOzOO5XoGFDkaUiuEBCbJPCoxr0"}
```

Envelope facts:

| Field | Meaning |
|---|---|
| `retries` | 2 — the native's internal attempts before success |
| `code` | 0 — success (the GPIA end-to-end error numbering) |
| `packageName` | `com.whatsapp` |
| `sha256` | SHA-256 of the declared APK (= `_is` from the integrity blobs) |
| `token` | a genuine **JWE**: the header `eyJhbGciOiJBMjU2S1ci…` = `{"alg":"A256KW","enc":"A256GCM"}` |
| `nonce` | **echo of the server's gpia nonce** — confirmation that the application put it into `IntegrityTokenRequest.setNonce` |

Token verification is performed by the WhatsApp server with Google (DECRYPTION): locally, the payload/verdict are not extracted from the JWE. The absence of the nonce inside the token in the open on the wire is the norm: it is bound into the token's requestHash.

## Difference from the registration gpia

| | Registration (/v2/code, /v2/register) | After login (socket) |
|---|---|---|
| JNI entry | `jvidispatchIIDOOOO`, `__WCAAPIGenerateGPIAParamOnMainImpl` (string `0x7cfd79751b`) | `jvidispatchIOOOO` id 3, `_WCAAPISendNonce` (`0x7cfdd3f360`) |
| Google API | Standard Integrity (`StandardIntegrityManager` + `PrepareIntegrityTokenRequest`, cloud project **293955441834**, `X/C40307Hsz`) | Classic Integrity (`IntegrityManagerFactory` + `setNonce`, cloud project **62949436**) |
| What is sent | the JSON `gpia`/`_gg`/`_gi` is encrypted with the AES envelope into an HTTP parameter | the raw encrypted envelope `{retries,code,packageName,sha256,token,nonce}` into the `<jws>` node |
| Request binding | requestHash = std-base64-nopad(authkey_raw 32B) (`X/J6X`, `C27011Jk` requires exactly 32B) | setNonce(the server nonce) |
| Warm token | `prepareIntegrityToken` in advance, rate limit 5/60 s ("GPIA_PREPARE_CALL") | a fresh `requestIntegrityToken` per nonce |

In other words: at registration WhatsApp hides the Play token deep inside an encrypted HTTP parameter together with `_gi`/`_gg`, and after login it answers the server nonce with a compact envelope whose main payload is the JWE token and the nonce echo.

## Timings and sizes of both sessions

| Session | nonce (server) | `<jws>` | Delay |
|---|---|---|---|
| 2.26.35.75, 02:48:52.736 → 02:48:57.266 | 57 B (76 b64url chars) | 2976 B (3968 b64 chars) | 4.53 s |
| 2.26.37.73 | 57 B | ≈2768 B (3692 chars) | ~4.5 s |

The stability of ~4.5 s across two versions is the time of Google Play services (the `Task` on `requestIntegrityToken`), not of the client.

## Where to look

- Dispatcher: `0x7cfdd1dd00`; id 3 branch `0x7cfdd1de70`, wrapper `0x7cfdd3f2c0`, stub `0x7cfdc78260`, body `0x7cfdd3f360`.
- Project preparation: `jvidispatchIIIIDOOO` `0x7cfdd21044` → `0x7cfdd31dd8`; the constants 62949436 / 855397460 / 796.6509679599703d.
- Play Core strings: rodata from `0x7cfd8ee24f`; function names `0x7cfd79767b` (`_WCAAPISendNonce`).
- Java: `X/C27871Nj.java` (~270–320), `X/C1HI.java` `A1J`, message 254, `X/C12G.java` `BD5`, the callback `APR` (`X/C31098Djv`).
- AB flags: 12964 (the first preparation argument before id 5/id 3), 12965 (registration mode).
- Related: `integrity-payload.md` (id 5), `timeline.md`.
