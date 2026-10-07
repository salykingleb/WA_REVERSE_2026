# tos_version — version of the accepted Terms of Service ("5")

## Position in the flow

A conditional POST /v2/register parameter (and also /v2/consent,
/v2/security, /v2/exist — the IF6.A0X method is called from all of them):
tells the server the version of the Terms of Service accepted by the client.
It is put as the literal `"5"` and only when TWO conditions hold — a
regional AB experiment and an accepted EULA. NOT a universal ToS acceptance
marker: for most countries the field is not sent at all.

## Format (+live example)

The string literal `"5"` (the ToS flow version; incremented by developers
when the terms change).

Live example (dump 2026-09-22, RU): the field is ABSENT from /v2/register.

## How it is formed in the app (class file:line, code)

```java
// X/IF6.java:624-630
public static final void A0X(IF6 if6, java.util.Map map) {
    C56371Pjt pjt = (C56371Pjt) C05D.A03(if6.A0Y);
    if (!pjt.A01() || GCN.A0U(pjt.A01).A06() <= 0) {
        return;                                  // NOT put
    }
    map.put("tos_version", B0W.A1Z("5"));
}
```

Condition (1) — AB flag 19561:

```java
// X/C56371Pjt.java:218-220
public final boolean A01() {
    return ((GD3) C05D.A03(this.A00)).A02(19561);
}
// X/PTP.java:434-441 — flag 19561 = "wamo_privacy_tos_reg_flow_enabled",
// Boolean, DEFAULT FALSE (X/C016407s.java:8894 builder.put(19561, false));
// enabled ONLY by experiments:
//   wamo_exp_test_{mx_co_id_br,pp}_tos_trigger_3_offline_android_*
// with targeting: country ∈ {ID, MX, BR, CO}, platform=android,
// app_version >= 2.25.29, windows ~November 2025 … 2027.
```

Condition (2) — the EULA acceptance time:

```java
// X/C02900De.java:96-102 — A06() = pref "eula_accepted_time" (long)
Ap6().getLong("eula_accepted_time", 0L)
// written >0 when the EULA is accepted on the first screen (EULA.java:310:
// A0R(System.currentTimeMillis()); by the time of register on a fresh
// install it is already done — but it does NOT decide without flag 19561).
```

## Value selection conditions

The field is sent ⇔ BOTH conditions are true:

1. the device got into the test group of AB experiment 19561
   (wamo_privacy_tos_reg_flow_enabled; geo ID/MX/BR/CO, android, app
   version ≥ 2.25.29; the control group and all other countries — off,
   default FALSE);
2. eula_accepted_time > 0 (EULA accepted).

Consequences:
- RU and most countries: tos_version is never sent (the live dump of
  2026-09-22 is consistent — the field is absent);
- within ID/MX/BR/CO — depends on bucketing: the control group does not send
  it either;
- there is a single value — "5"; there are no other literals in the code.

## Cryptography and encoding

- No cryptography: literal → UTF-8 → percent-encoding (the digit does not
  change).

## Relation to the <iq tos> notice 20210210 after login

tos_version in /v2/register is NOT the same thing as the post-login ToS
notice `<iq tos>` (id "20210210"). The post-login mechanism:

```java
// X/C42801ux.java:76-81 — the list of ToS-notice versions:
List listSingletonList = Collections.singletonList("20210210");   // this.A0A

// X/C31350DoL.java:33-35 — whether to show the ToS dialog after login:
return c016407s.A0w(791)                          // AB flag 791 (tos-notice)
       && C42801ux.A00(ux).A00("20210210") == 2;  // state 2 = accepted
```

That is, the string "20210210" is the identifier of the current revision of
the terms in the post-login flow (state: 0=not shown, 2=accepted; a check
for ==2). A0X in register sends "5" — an independent numeric version of the
same product flow (wamo_privacy_tos_reg_flow), tied to the In-Registration
ToS screen rather than to the iq notice. Indirect link: a non-empty list of
accepted ToS versions (A02(if6).A0C()) is the condition for a non-zero
entrypoint=suma in A0T.

## Examples

- RU, live dump: tos_version absent (correct behavior for cc=7).
- A device in ID/MX/BR/CO, test group, EULA accepted: `tos_version=5`.
- The same device, control group: tos_version absent.
