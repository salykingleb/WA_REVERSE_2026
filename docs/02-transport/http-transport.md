# HTTPS_transport — common framework of the /v2/code and /v2/register registration HTTP requests

## Place in the flow

WhatsApp Android 2.26.35.75 (versionCode 263507522) registration is performed with two
HTTP requests to the host `v.whatsapp.net`:

1. `POST https://v.whatsapp.net/v2/code?<query of 50 parameters>` — request for the SMS/voice verification code;
2. `POST https://v.whatsapp.net/v2/register?<query of 49 parameters>` — activation of the number with the entered code (`code=111111` in the live capture).

Both requests are HTTP/1.1 over TLS, the body is `ENC=<...>&H=<...>`. Assembly and sending
are managed by `X/IF6.java` (RegistrationHttpManager). The app has two sending paths,
the choice between which is made by AB flags 24762/24763:

- **Kotlin path (new)**: `X/IF6.java` → `com/whatsapp/registration/core/http/KotlinRegistrationBridge.java` →
  `com/whatsapp/registration/core/http/retry/RetryingHttpClient.A00` (the class failed to decompile
  in jadx, analyzed via the full dex disassembly `out\jadx_badcode\classes9.dump`) →
  `X/I1G.java` (RegistrationHttpClient.executePost) → `X/H0O.java` (TigonWaHttpClient) → Tigon.
  Here the construction of the `ENC=…&H=…` body, the Authorization header and the final URL is fully visible.
- **wamsys-native path (old)**: `X/IF6.java` → `X/I7I.A00` + HCx runners →
  `com/whatsapp/wamsys/JniBridge.jvidispatchIOOOOOOOOO(...)` (native libs.so) →
  msys UrlRequest → the `X/Iqs` bridge (dexdump, lost in jadx) → the same Tigon, but with the
  `User-Agent` and `WaMsysRequest: 1` headers added first by the bridge.

Both paths produce the same wire format (confirmed by a live capture).

## Wire format (type, encoding, length, LIVE example from the capture)

**Request line and URL (live capture 2026-09-22, Samsung SM-A325F, Wi-Fi without SIM):**

```
POST /v2/code?method=sms&backup_token=…&_gs=…&sim_mnc=000&id=…&mnc=000&_gg=…&network_radio_type=1&lg=ru&rc=0&pid=28104&…&lc=RU HTTP/1.1
POST /v2/register?code=111111&backup_token=…&sim_mnc=000&_gs=…&id=…&device_ram=3%2C56&…&lc=RU HTTP/1.1
```

The full order of the 50 /v2/code fields and the 49 /v2/register fields — see params-order.md.
All values are percent-encoded (`%XX` with UPPERCASE hex; `device_ram=3%2C56`, comma → `%2C`).

**Exact order of HTTP headers (live capture):**

```
User-Agent: WhatsApp/2.26.35.75 Android/13 Device/samsung-SM-A325F
WaMsysRequest: 1
Authorization: <base64.std(DER-конкатенация цепочки сертификатов, см. отдельный README)>
request_token: <UUID в верхнем регистре>
Content-Type: application/x-www-form-urlencoded
Content-Length: <длина тела в байтах>
Host: v.whatsapp.net
Connection: Keep-Alive
Accept-Encoding: gzip
```

`Authorization` is present only when keystore attestation is enabled; in the native
(msys) order it stands right after `WaMsysRequest`, in the Kotlin map — after `Content-Length`.

**Body (form-urlencoded, one byte-exact format):**

```
ENC=<base64.RawURL(pub32 || AES-256-GCM(query) || tag)>&H=<base64.RawURL(DER-ECDSA-подпись)>
```

**Double submission of parameters** — the key feature of the protocol: one and the same
query string is sent TWICE within a single request:
1. in plaintext in the URL after `?` (live capture: both the `/v2/code?…` and `/v2/register?…` lines contain the full set of parameters);
2. encrypted as that same string inside `ENC` in the body.

The ENC plaintext is byte-for-byte equal to the query string from the URL (for /v2/code this is confirmed
by the code structure: the query variable is used both in the URL and as the encryption input).

## How it is built in the app (Java/Kotlin class file:line, code fragments)

**Base URL.** `X/C0UA.java:45`:
```java
public static final String A0Y = A00("zffba(==d<ezsfasbb<|wf");  // XOR-18 → "https://v.whatsapp.net"
```
The decryption method is an XOR of each character with 18 (the same scheme as in the string pool below).

**Endpoints.** `X/AbstractC40561Hxd.java` — a pool of obfuscated XOR-18 strings
(the decoder at :27–32: `sb.append((char)(str.charAt(i) ^ 18))`):
```java
A07 = A00("=d =q}vw");      // "/v2/code"     (строка 7)
A0C = A00("=d =`wu{afw`");  // "/v2/register" (строка 11)
A0E = "/v2/exist";          // там же: /v2/consent, /v2/challenge, /v2/security, /v2/autoconf
```
Manual XOR-18 check: `'='(0x3D)^18=0x2F='/'`, `'d'(0x64)^18=0x76='v'`, `' ' (0x20)^18=0x32='2'` → `/v2/code`.

**The "ENC"/"H" strings.** The same class, :23–24: `A05 = A00("W\\Q") = "ENC"`, `A09 = A00("Z") = "H"`
(`'W'(0x57)^18=0x45='E'`). For the native path they are passed through `X/AbstractC39157HWs.java`
(`A02="ENC"`, `A03="H"`, `A00` → provider C13D id 2874, `A01` → keystore provider id 6015).

**URL and body assembly.** RetryingHttpClient.A00 (dexdump classes9.dump, offset 0x0092–0x0249):
URL = `baseUrl + endpoint + "?" + query` (the literal `"?"` = string@1129); under the
HOST/DOMAIN_FRONTING/PROXY strategies the URL is parsed via `URI.create(url).getRawQuery()/getRawPath()`
(offsets 0x00b2–0x00be, 0x0149–0x0159) to rewrite Host/`X-Forwarded-Host`. The body =
`"ENC=" + enc + "&" + "H=" + h` (offsets 0x01f4–0x02b6).

**User-Agent.** `X/C0V2.java`:
- :16 — the version `public static final String A09 = "2.26.35.75".replace(' ', '_');`
- :108–113 — versionCode: business build → `"1062737884"`, otherwise `"263507522"`;
- :115–163 — assembly of the A01 string:
```java
Pattern patternCompile = Pattern.compile("[^,\\.\\w\\-\\(\\)]");   // :118
strReplaceAll  = patternCompile.matcher(Build.VERSION.RELEASE).replaceAll("_");      // :130
strReplaceAll2 = patternCompile.matcher(Build.MANUFACTURER).replaceAll("_");         // :136
strReplaceAll3 = patternCompile.matcher(Build.MODEL).replaceAll("_");                // :142
// итог: "<App>/2.26.35.75; Android/<RELEASE> Device/<MANUFACTURER>-<MODEL>;;"
```
Wire format: `WhatsApp/<version> Android/<RELEASE> Device/<MANUFACTURER>-<MODEL>`.
The Kotlin path puts the UA as the first header (`H0O.java:87` `tigonRequestBuilderA01.addHeader("User-Agent", this.A02.A03())`).

**WaMsysRequest: 1.** The msys→Tigon bridge `X/Iqs` (dexdump classes9.dump ~3 695 000):
`TigonRequestBuilder(method,url).addHeader("User-Agent",UA).addHeader("WaMsysRequest","1")`,
then the headers from the native map.

**Content-Type / Content-Length.** `X/I1G.java:26–27`:
```java
linkedHashMapA0O.put("Content-Type", "application/x-www-form-urlencoded");
linkedHashMapA0O.put("Content-Length", String.valueOf(length));
```
In the wamsys path Content-Type is set by the native code, Content-Length is computed by Tigon itself.

**Host.** Tigon sets it automatically from the URL; additionally it is set explicitly under
domain-fronting (see above); in normal mode — `Host: v.whatsapp.net`.

**Connection / Accept-Encoding.** `X/H0O.java:126`: `addHeader("Accept-Encoding","gzip")`;
`Connection: Keep-Alive` is the default of Tigon's HTTP/1.1 stack.

**DNS pinning.** `X/O90.java:795` pins the DNS records `v.whatsapp.net` and `v.whatsapp.net.`.

## Native implementation (if involved)

- **Stack — Tigon**: `com/crossapp/tigonhttp/TigonHttpClient*.java` (native
  `tigonhttpclient-jni`, `mnscertificateverifier`), folly/facebook-tigon; libs.so
  is linked with `libtigon-ue-reporter.so`. Both paths go through
  `com.facebook.tigon.iface.TigonRequestBuilder`.
- **ALPN/HTTP version**: `TigonHttpClientConfig.useALPNProtocolsFromMNSTLSContext` —
  the ALPN list is taken from the MNS TLS context; `forceHttp2` exists but is off
  by default. In the live capture with v.whatsapp.net `http/1.1` was negotiated; the requests go over HTTP/1.1.
- **TLS ClientHello** (native TLS, not visible in Java/dex; taken from a live client of the same stack):
  - 17 ciphers in order: `1301,1302,1303,c02b,c02c,cca9,c02f,c030,cca8,c009,c00a,c013,c014,009c,009d,002f,0035`;
  - extensions in order: `server_name(SNI), extended_master_secret, renegotiation_info,
    supported_groups(x25519,p256,p384), ec_point_formats, session_ticket, ALPN,
    status_request(OCSP), signature_algorithms(9), key_share(x25519), psk_key_exchange_modes(DHE),
    supported_versions(TLS1.3+TLS1.2), padding`;
  - padding of the ClientHello record to 512 bytes (AlwaysAddPadding, BoringSSL style — Tigon
    is built on BoringSSL/fizz);
  - **JA3 of the fresh (no PSK) ClientHello = `9b02ebd3a43b62d825e1ac605b621dc8`**. Origin:
    the fingerprint of the live WhatsApp client on the Tigon stack. With TLS resumption the real client
    adds a PSK extension, and the JA3 changes; the first request of a session is the fresh variant.
- **wamsys path**: JniBridge `jvidispatchIOOOOOOOOO` goes into libs.so (strings are obfuscated);
  the `request_token` header is generated there as well.

## Conditions for choosing values (variants, ranges, when absent)

- Endpoint: `/v2/code` when requesting the code, `/v2/register` on confirmation; neighboring
  endpoints of the same manager: `/v2/exist`, `/v2/consent`, `/v2/challenge`, `/v2/security`, `/v2/autoconf`.
- The choice of the Kotlin/wamsys path — AB flags 24762/24763; the wire format does not change because of it.
- `Authorization` is absent if attestation is off (AB flag 1934, `C13D.A06()`).
- `WaMsysRequest: 1` — a constant, only the wamsys path adds it explicitly; in the capture it is always present.
- The HTTP method is always POST, the body encoding is always `application/x-www-form-urlencoded`.

## Cryptography and encoding

- URL/query: percent-encoding, UPPERCASE hex (`%2C`, `%7B`, `%2F`); byte parameters —
  RFC3986-unreserved as is (see params-order.md, `X/HSJ.A00`).
- ENC: X25519 + AES-256-GCM, base64.RawURL — in detail in ENC.md.
- H: DER ECDSA-P256-SHA256, base64.RawURL — in detail in H.md.
- Authorization: base64.Std (flag 2, NO_WRAP) of the DER concatenation — Authorization.md.
- Transport: TLS 1.3/1.2 (supported_versions), HTTP/1.1 over ALPN.

## Examples

A live skeleton of the /v2/code request (abbreviated, real values from the 2026-09-22 capture):

```
POST /v2/code?method=sms&backup_token=<20B-urlenc>&_gs=<…>&sim_mnc=000&id=<20B-urlenc>&mnc=000&_gg=<…>&network_radio_type=1&lg=ru&rc=0&pid=28104&_ge=%7B%22sb%22%3Afalse%2C%22sv%22%3Afalse%7D&cellular_strength=5&gpia=<…>&hasinrc=1&aid=<…>&mcc=000&_gp=<…>&hasav=2&device_ram=3%2C56&db=1&sim_type=0&prefer_sms_over_flash=false&access_session_id=<…>&sim_mcc=000&roaming_type=0&mistyped=7&airplane_mode_type=0&e_ident=<…>&e_skey_sig=<…>&entrypoint=suma&token=<…>&in=9206309125&expid=<…>&simnum=0&cc=7&education_screen_displayed=false&e_skey_val=<…>&authkey=rsgyL8y0zyOXIJT4KofOJGJaxrlKUp6TuBsPvGzZAzs&e_keytype=BQ&e_regid=extZ9w&client_metrics=<…>&_gi=<…>&e_skey_id=E-mO&feo2_query_status=client_capabilities_cached&reason=&fdid=da1e6667-b5ed-4fa6-9828-53283393c119&recaptcha=%7B%22stage%22%3A%22ABPROP_DISABLED%22%7D&_ga=<…>&lc=RU HTTP/1.1
User-Agent: WhatsApp/2.26.35.75 Android/13 Device/samsung-SM-A325F
WaMsysRequest: 1
Authorization: <base64 std, ~4-5 KB цепочки root→TEE-CA→device-TEE→leaf>
request_token: <UUID uppercase>
Content-Type: application/x-www-form-urlencoded
Content-Length: <N>
Host: v.whatsapp.net
Connection: Keep-Alive
Accept-Encoding: gzip

ENC=<base64url>&H=<base64url>
```

For /v2/register the first query fields: `code=111111&backup_token=<the same 20B>&sim_mnc=000&_gs=<NEW>…` —
id/backup_token/aid/_gp/_ga/access_session_id/fdid/expid/e_*/authkey/lg/lc/rc/mcc/mnc/sim_*/device_ram/db/hasinrc/pid
are carried over from /v2/code, while `_gs, _gg, gpia, _gi, client_metrics` are created fresh.
