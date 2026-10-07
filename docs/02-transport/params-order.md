# params_order — exact sequence of query fields for /v2/code and /v2/register

## Place in the flow

The order of fields in the query string of the registration requests is neither alphabetical nor random:
it is the fixed **native sequence of the parameter builder** (msys),
observed on the wire. The string with this order is sent twice: openly in the URL
(`POST /v2/code?…`) and encrypted inside `ENC` (the envelope's plaintext is byte-for-byte
equal to the query string — see ENC.md). Reproducing the order is critical:
the ENC plaintext must match the URL part, and the server's plausibility checks
may compare the sequence against the client reference.

## Wire format (type, encoding, length, LIVE example from the capture)

**/v2/code — 50 parameters, the live order (capture 2026-09-22, SM-A325F):**

```
1 method, 2 backup_token, 3 _gs, 4 sim_mnc, 5 id, 6 mnc, 7 _gg, 8 network_radio_type,
9 lg, 10 rc, 11 pid, 12 _ge, 13 cellular_strength, 14 gpia, 15 hasinrc, 16 aid,
17 mcc, 18 _gp, 19 hasav, 20 device_ram, 21 db, 22 sim_type, 23 prefer_sms_over_flash,
24 access_session_id, 25 sim_mcc, 26 roaming_type, 27 mistyped, 28 airplane_mode_type,
29 e_ident, 30 e_skey_sig, 31 entrypoint, 32 token, 33 in, 34 expid, 35 simnum,
36 cc, 37 education_screen_displayed, 38 e_skey_val, 39 authkey, 40 e_keytype,
41 e_regid, 42 client_metrics, 43 _gi, 44 e_skey_id, 45 feo2_query_status, 46 reason,
47 fdid, 48 recaptcha, 49 _ga, 50 lc
```

**/v2/register — 49 parameters, the live order:**

```
1 code, 2 backup_token, 3 sim_mnc, 4 _gs, 5 id, 6 device_ram, 7 db, 8 _gg,
9 sim_operator_name, 10 passkey_login_status, 11 lg, 12 rc, 13 network_radio_type,
14 pid, 15 _ge, 16 cellular_strength, 17 gpia, 18 hasinrc, 19 aid,
20 network_operator_name, 21 mcc, 22 _gp, 23 mnc, 24 entered, 25 access_session_id,
26 sim_mcc, 27 roaming_type, 28 mistyped, 29 airplane_mode_type, 30 has_play_store,
31 sim_type, 32 e_ident, 33 e_skey_sig, 34 entrypoint, 35 in, 36 expid, 37 simnum,
38 cc, 39 e_skey_val, 40 authkey, 41 e_keytype, 42 e_regid, 43 _gi, 44 client_metrics,
45 e_skey_id, 46 fdid, 47 recaptcha, 48 _ga, 49 lc
```

Both orders were verified independently against the live wire: 50/50 and 49/49 fields, field by field.

The separator is `&`, a `k=v` pair, the values encoded with percent-encoding with UPPERCASE
hex (`%2C`, `%7B`, `%2F`, `%2B`, `%3D`); across the entire live dump there is not a single
lowercase `%xy`. A live fragment of the beginning of /v2/code:

```
method=sms&backup_token=<20B>&_gs=<b64url>&sim_mnc=000&id=<20B>&mnc=000&_gg=<b64url>&network_radio_type=1&lg=ru&rc=0&pid=28104&…
```

## How it is built in the app (Java/Kotlin class file:line, code fragments)

- **Order storage** — `X/C40787I4g.java` (RegistrationRequestBuilder):
  a LinkedHashMap "name → value"; the order of the pairs on the wire = the insertion
  order, NOT sorting.
- **Serialization** — the mapper IuA-46 (dexdump classes9.dump 0x19d6–0x1a02):
  ```
  URLEncoder.encode(key, UTF-8) + "=" +
      (ключ ∈ множеству A01 ? value : URLEncoder.encode(value, UTF-8))
  ```
  The A01 set — the keys whose values are already encoded in advance (byte parameters,
  encoded on insertion via `X/HSJ.A00`).
- **Encoding of byte values** — `X/HSJ.A00`: RFC3986-unreserved
  `A-Za-z0-9-._~` are passed as is, the remaining bytes → `%XX` with UPPERCASE hex.
  This is how id, backup_token, authkey, e_*, expid, access_session_id,
  fdid-like byte blobs and pre-encoded JSON parameters are encoded.
- **Assembly of the request as a whole** — `X/IF6.java` (RegistrationHttpManager) fills
  the builder; the endpoint-specific fields are added by
  `com/whatsapp/registration/core/http/KotlinRegistrationBridge.java`.
  In the wamsys path the same is done by the native msys builder (libs.so) — the live order
  coincides for both paths.
- **Substitution into the URL and into ENC**: dexdump of RetryingHttpClient.A00 — the query variable
  is used both as the `"?" + query` URL suffix and as the ENC encryption plaintext.

## Native implementation (if involved)

The final gluing of the query and the on-wire order in the wamsys path are defined by the native msys code
of libs.so (obfuscated; the order was reconstructed from the live capture and coincides with the
Java path). No separate exported `_WCAPI*` functions for query assembly were
identified — the order is formed by the insertion LinkedHashMap structure +
endpoint-specific insertions.

## Conditions for choosing values (variants, ranges, when absent)

**The difference between /v2/code and /v2/register (per the live capture):**

- Only in /v2/code (6 fields): `method` (sms/voice), `hasav` (=2 on the live wire),
  `prefer_sms_over_flash` (false), `token` (the setup HMAC token),
  `education_screen_displayed` (false), `feo2_query_status`
  (client_capabilities_cached), `reason` (empty).
- Only in /v2/register (6 fields): `code` (the entered code, live 111111),
  `sim_operator_name` (empty), `network_operator_name` (empty),
  `passkey_login_status` (not_attempted), `entered` (=1), `has_play_store` (true).
- Created fresh for each of the two requests: `_gs`, `_gg`, `gpia`, `_gi`,
  `client_metrics` (including the `_gg` JWS token in register — a DIFFERENT one, fresh per request;
  `client_metrics.attempts` in code = 7, in register = 0).
- Carried over unchanged from /v2/code to /v2/register: id, backup_token, aid, _gp,
  _ga, access_session_id, fdid, expid, e_* (all the signal ones), authkey, lg, lc, rc,
  mcc, mnc, sim_mcc, sim_mnc, device_ram, db, hasinrc, pid.
- Absent in BOTH live requests: `advertising_id`, `tos_version`,
  `clicked_education_link`, `vname` (consumer build), `method` in register.
- Live environment values of that session: mcc/mnc/sim_mcc/sim_mnc = 000 (Wi-Fi without
  SIM), sim_type=0, simnum=0, cellular_strength=5, network_radio_type=1 (WiFi),
  roaming_type=0, airplane_mode_type=0, mistyped=7 in both, entrypoint=suma in both,
  device_ram=3%2C56, hasinrc=1, db=1, pid=28104, in=9206309125, cc=7, lg=ru, lc=RU, rc=0.

**Why this order matters:**

1. The invariant "URL query == plaintext ENC" requires identical assembly in both places.
2. The ENC plaintext is signed (H) — byte similarity affects the signature.
3. The server parses the pairs regardless of order (semantically), but plausibility scoring
   of fresh installs may compare the sequence/composition against the client reference;
   extra trailing keys (e.g. `advertising_id` at the end) are an anomaly against
   the live wire, where they are absent.

## Cryptography and encoding

- Keys: always ASCII literals (`method`, `_gs`, …), URLEncoder does not change them.
- Values:
  - byte ones — HSJ/RFC3986, %XX UPPERCASE, unreserved raw;
  - strings — Java `URLEncoder.encode` (form style: space → `+`, `*` is not escaped,
    `~` → `%7E`); in the live data there are no such characters, all `%XX` are UPPERCASE;
  - JSON parameters (`_ge`, `client_metrics`, `recaptcha`, `_ga`, `_gs` wrappers)
    are encoded as a whole: `%7B` `{`, `%22` `"`, `%3A` `:`, `%2C` `,`, `\/` inside `_ga` → `%5C%2F`.
- No sorting whatsoever: the pure insertion order of the builder.

## Examples

Live order-marker values (capture 2026-09-22):

```
/v2/code:    …&e_ident=<…>&e_skey_sig=<…>&entrypoint=suma&token=<…>&in=9206309125&…&e_regid=extZ9w&client_metrics=%7B%22attempts%22%3A7…&_gi=<…>&e_skey_id=E-mO&feo2_query_status=client_capabilities_cached&reason=&fdid=da1e6667-b5ed-4fa6-9828-53283393c119&recaptcha=%7B%22stage%22%3A%22ABPROP_DISABLED%22%7D&_ga=<…>&lc=RU
/v2/register:…&e_regid=extZ9w&_gi=<NEW>&client_metrics=%7B%22attempts%22%3A0…&e_skey_id=E-mO&fdid=da1e6667-b5ed-4fa6-9828-53283393c119&recaptcha=<…>&_ga=<SAME>&lc=RU
```

Note the permutation inside the common signature: in /v2/code the order is
`client_metrics → _gi → e_skey_id`, in /v2/register — `_gi → client_metrics →
e_skey_id`; this is NOT sorting, but different insertion points in the builders of the two endpoints.
