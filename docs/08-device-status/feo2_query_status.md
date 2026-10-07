# feo2_query_status — FEO2 client capabilities query status ("client_capabilities_cached" | "success_get_client_capabilities" | …)

## Purpose

A status string about how the "client capabilities" request via FEO2 ended (or never started) — the Facebook Event SDK client (a call to the `com.facebook.services` service for the capabilities blob, used in autoconf verification). Shows the server whether the client had an up-to-date state of the autoconf subsystem by the moment of the code request.

## Place in the flow

| Request | Presence |
|---|---|
| /v2/code | yes (the main path: `RequestCodeRepository$requestCode$2.java:403–406`) |
| /v2/register | **no** |
| /v2/exist | yes (`IF6.A0j:1405–1408`) |

The value is the SharedPreferences key `pref_autoconf_feo2_query_status`, **default "did_not_query"** (reset to the default: `X/C38467Gys.java:225`).

## Format

A literal string:

```
feo2_query_status=client_capabilities_cached        ← живой пример (SM-A325F, только в /v2/code)
```

## How it is formed in the application

Written by the autoconf manager `X/C40776I3l.java` (AutoconfManager), method `A01()` — acquireClientCapabilities:

```java
if (this.A01 != null) {                     // capabilities уже в кэше
    prefs → "pref_autoconf_feo2_query_status" = "client_capabilities_cached";
} else {
    // запрос через C39541Hfp (typed contract FeO2ClientTypedContract_Query)
    // к провайдеру com.facebook.services (C44344Jo1 builder с FB-подписями)
    bundle.putBoolean("useDebugKey", false); ... "query" ...
    this.A01 = bundle.getByteArray("capabilities");
    prefs = (this.A01 == null)
        ? "success_null_client_capabilities"
        : "success_get_client_capabilities";
    // при исключении — error_* (см. таблицу)
}
```

Request mechanics: the typed contract `FeO2ClientTypedContract_Query` is assembled toward the package `com.facebook.services` (Facebook signature validation — the `C44344Jo1` builder with `ImmutableSetMultimap("com.facebook.services", signatures)`), a call with the method `"query"`, and `byte[] capabilities` is taken from the response. The operation result is written into the pref `pref_autoconf_feo2_query_status`, from which the /v2/code and /v2/exist maps read it.

What FEO2 is: the Facebook Event SDK client (FeO2 = a Facebook Events provider v2 contract) — the channel for obtaining the device's "client capabilities" via Meta services installed on GMS devices. The presence of a successful status is a sign of a device with the full Google/Meta service stack.

## Value selection conditions (the full palette of ~9 values)

| Value | Condition |
|---|---|
| **"client_capabilities_cached"** | capabilities are ALREADY in the process memory (the A01 cache != null) — a repeated request in the same session/process; live dump |
| **"success_get_client_capabilities"** | the request to com.facebook.services succeeded, the capabilities were obtained (byte[] != null) |
| "success_null_client_capabilities" | the request succeeded, but capabilities == null (the provider returned no data) |
| "error_remote_exception" | RemoteException from the provider |
| "error_wrapped_provider_exception" | a wrapped provider exception (C38813HGz) |
| "error_illegal_argument_exception" | IllegalArgumentException |
| "error_illegal_state_exception" | IllegalStateException |
| "error_security_exception" | SecurityException (including all other exceptions) |
| "did_not_query" | the default: by the moment of the code request the FEO2 request has not been performed yet (a fresh installation before autoconf completes) or has been reset (C38467Gys:225) |

## Order of values on the first/repeated request

- **A fresh installation, autoconf has not worked yet**: the pref is at the default → /v2/code goes with `did_not_query` (or one of the error_*, if the first attempt failed).
- **The first successful FEO2 request**: pref ← `success_get_client_capabilities` (or success_null_client_capabilities) → subsequent requests carry it.
- **A repeated request in the same process** (the cache already in the A01 memory): pref ← `client_capabilities_cached` — the dominant value on devices that have passed autoconf (live dump).
- **Reset**: on certain flow transitions (C38467Gys:225) the pref returns to `did_not_query`.

That is, over time the values move: did_not_query → (success_* | error_*) → client_capabilities_cached → (a reset to did_not_query is possible).

## Examples

- Live dump (SM-A325F, autoconf worked earlier): `feo2_query_status=client_capabilities_cached` — only in /v2/code.
- The first code request right after installation: `feo2_query_status=did_not_query`.
- A device without com.facebook.services where the contract failed: `feo2_query_status=error_wrapped_provider_exception`.
