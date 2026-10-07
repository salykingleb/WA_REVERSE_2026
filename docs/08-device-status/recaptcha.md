# recaptcha — reCAPTCHA flow JSON state ({"stage":"ABPROP_DISABLED"})

## Purpose

A nested JSON reflecting the state of the client reCAPTCHA mechanism (Google reCAPTCHA Enterprise, integrated through Google Play Services): whether the check was enabled by an AB flag, at which stage the initialization/token fetching is, whether there is a token, its length and age. With the AB flag off — the standard state of a regular registration — the JSON degenerates to `{"stage":"ABPROP_DISABLED"}`.

## Place in the flow

| Request | Presence |
|---|---|
| /v2/code | yes |
| /v2/register | yes |
| /v2/exist | yes |

Placed by the method `IF6.A0q()` (`X/IF6.java:975–1054`, a smali dump — the method did not decompile into Java), which reads the reCAPTCHA client wrapper `C40323HtF` (RecaptchaClientHandler) and builds a JSONObject.

## Format (+live example)

A JSON string; the minimal form — only stage, the extended one — the token and metrics:

```
recaptcha={"stage":"ABPROP_DISABLED"}        ← живой пример: ОБОИХ запросах (SM-A325F)
```

Extended form (with reCAPTCHA enabled):

```
{"stage":"FETCH_SUCCEEDED","token":"<обрезка до 7500>","token_length":1234,"token_age":45678}
{"stage":"INIT_FAILED","error":"java.lang.Exception: No internet"}
```

## How it is formed in the application

`IF6.A0q()` builds the JSON from the fields of `C40323HtF` (`X/C40323HtF.java`):

| JSON field | Source |
|---|---|
| token | `C35211hE.A00[0]` — the token string from the pref `less_beep_beep_identi`, truncated by `AbstractC42301u7.A11(token, 7500)`; the key is set only if there is a token |
| token_length | `GCM.A0c(token)` — the length |
| token_age | `now − less_beep_beep_time` (ms; `AbstractC151086j4.A18`) — the token age |
| error | `exception.toString()` on an exception |
| stage | the HFB enum — always |

### Stages — the enum `X/HFB.java:20–38` (9 values)

| stage | When |
|---|---|
| ABPROP_NOT_CHECKED | the initial value before the first access to the gate |
| **ABPROP_DISABLED** | the AB permission did NOT come up (the standard for registration) |
| ABPROP_ENABLED | the AB permission came up |
| INIT_STARTED | start of reCAPTCHA client initialization |
| INIT_FAILED | initialization failed (no internet, an exception, an ineligible country) |
| INIT_SUCCEEDED | the client is initialized |
| FETCH_STARTED | start of token fetching |
| FETCH_FAILED | token fetching failed |
| FETCH_SUCCEEDED | the token was obtained |

### The AB flag gate — `C40323HtF.A02()` (`X/C40323HtF.java:101–121`)

```java
if (iA05 < 0) {
    iA05 = AbstractC04780Ln.A01.A05(1, 1000);                    // случайное 1..1000
    AbstractC203528ts.A1D(prefs, "more_sheep_random_number", iA05); // персист на устройстве
}
Boolean enabled = iA05 < this.A0A.A0Y(7343);                      // сравнение со значением AB-пропа 7343
this.A02 = enabled ? HFB.A03 /*ABPROP_ENABLED*/ : HFB.A02 /*ABPROP_DISABLED*/;
```

- On the device a number 1..1000 is generated once and saved (pref `more_sheep_random_number` — a user "caught" into the experiment);
- it is compared with the threshold of AB prop 7343: less — enabled (ABPROP_ENABLED), otherwise disabled (ABPROP_DISABLED);
- the result is fixed for the entire process lifetime.

With ABPROP_ENABLED the initialization goes through `HOm.A00(application, "6LcgaR4pAAAAAFMQmjEQyA7UegLcjegCi241YDXv")` — the reCAPTCHA site key; the SIM country `gb` (GCQ.A0d) is marked ineligible ("makes recaptcha unusable" → INIT_FAILED with an ineligible country error).

When the parameter changes: the stage lives in the wrapper and updates as the flow progresses (INIT_* → FETCH_*); token/token_length/token_age appear only after FETCH_SUCCEEDED. In a typical registration the AB flag is off → all requests of the session carry the unchanging `{"stage":"ABPROP_DISABLED"}` (without token/error fields).

## Value selection conditions

| stage | Condition |
|---|---|
| **ABPROP_DISABLED** | the persistent random number (1..1000) >= the threshold of AB prop 7343 — the standard registration state; JSON without the other fields |
| ABPROP_NOT_CHECKED | the gate has not been computed in the process yet (in practice not seen on the wire — A0q is called after the initialization) |
| ABPROP_ENABLED → INIT_STARTED | the gate passed, the initialization began |
| INIT_FAILED | no network at start / an initialization exception / SIM country gb |
| INIT_SUCCEEDED → FETCH_STARTED | the client is ready, token request |
| FETCH_FAILED | the token was not obtained (the error in the JSON field error) |
| FETCH_SUCCEEDED | the token was obtained: the JSON contains token (truncated to 7500), token_length, token_age |

## Examples

- Live dump: `recaptcha={"stage":"ABPROP_DISABLED"}` — byte-for-byte identical in /v2/code and /v2/register.
- A device in the experiment after a successful token fetch: `{"stage":"FETCH_SUCCEEDED","token":"...","token_length":1042,"token_age":31200}`.
- Network loss during initialization: `{"stage":"INIT_FAILED","error":"..."}`.
