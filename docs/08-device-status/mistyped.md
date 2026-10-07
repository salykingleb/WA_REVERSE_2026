# mistyped — code of the phone number entry method on the registration screen

## Purpose

A scalar code "snapshot" of HOW the user entered their number on the entry screen (RegisterPhone) relative to the SIM number and UI events: with autofill or manually, whether the input matched the SIM number, whether there was an edit, whether there was phone permission. Lets the server assess the typical pattern of "a human types the number themselves" versus "autofill/substitution".

The name is misleading: this is not "the number of typos" but a code of a branch of the number entry UI state machine.

## Place in the flow

| Request | Presence |
|---|---|
| /v2/code | yes (if the value in the pref != null — practically always) |
| /v2/register | yes (the same pref, the same value) |

The value is computed ONCE during number entry, saved into SharedPreferences, and then **the same value** is read by both requests of the session:

- /v2/code: `RequestCodeRepository$requestCode$2.java:195` → `IF6.A0H` (`X/IF6.java:1110`) — put if != null;
- /v2/register: `VerifyCodeRepository$verify$2` → `IF6.A0I` (`X/IF6.java:365`) — put if != null.

## Format

A number-string "0".."7" (plus "-1" in a special support path):

```
mistyped=7        ← живой пример: ОДНО значение "7" в ОБОИХ запросах (SM-A325F)
```

## How it is formed in the application

The generator — `RegisterPhone.A0z(String str, String str2, int i)` (`com/whatsapp/registration/app/phonenumberentry/RegisterPhone.java:715–752`). Called on number entry UI events; the result is immediately persisted into the pref:

```java
String strA0z = registerPhone.A0z(strA00, strA01, i);
AbstractC49522Fq.A1O(GCN.A0B(...), "com.whatsapp.registration.RegisterPhone.mistyped_state", strA0z);
```
(`RegisterPhone.java:1283–1284`)

The logic of A0z (inputs: str — the entered number, str2 — the SIM number/substitution, i — the UI event code):

```java
if (!super.A0S.A0I()) return "7";              // phone-permission НЕ выдан
boolean zA1Y = A1Y(IEz.A0H(...));              // сработало автозаполнение номера
String str3 = this.A0O;                        // SIM-номер устройства
if (str3 == null) return "6";                  // SIM-номера нет
if (!A1y && !A1x && !zA1Y && !this.A0t) return "6";   // не было ни автозаполнения, ни правок
boolean z = !zA1Y && IEz.A00(digits(str2), digits(str3)) == 0;  // ввод == SIM-номер (оба ≥ 6 цифр)
if (i == 30) { ... "0"/"3"/"4"/"5" }           // событие 30: ввод/автозаполнение
if (i == 31) { ... "1"/"2" }                   // событие 31: правка номера
if (i == 32) { ... "1"/"2"/"5" }               // событие 32: правка номера
return "5";                                    // прочее
```

Conditional flags:
- `A0S.A0I()` — whether the READ_PHONE_STATE permission is granted (the phone-permission helper);
- `this.A0O` — the read SIM number (the same source as for simnum);
- `zA1Y` — the number field autofill flag (from SIM/account);
- `A0l` — the flash-call eligibility indicator of the screen;
- `A0t` — the UI flag of screen activity;
- `z` — an exact match of the entered number with the SIM number (after stripping non-digit characters, `IEz.A00` requires a length >= 6 for both);
- `i` — the UI event code (30/31/32 — different actions with the input field).

The separate value "-1" is set in a special path — a repeated checkIfExists from SupportForm (not the main flow).

## Value selection conditions

| Value | Condition |
|---|---|
| **"7"** | phone permission NOT granted — the application cannot even read the SIM number. The dominant case for "clean" registrations without granted permissions |
| **"6"** | permission granted, but: SIM number == null (no SIM / the operator does not return it), OR there was neither autofill, nor edits, nor activity |
| "0" | event 30, flash-eligible (A0l), UI active (A0t), input matched the SIM (z) |
| "3" | event 30, A0l, not A0t: input matched the SIM AND there was autofill |
| "4" | event 30: autofill worked (zA1Y) |
| "5" | event 30: the remaining cases (input did not match the SIM etc.); also the default of the other branches |
| "1" | event 31/32, UI active (A0t) — number edit |
| "2" | event 31/32, not A0t — number edit |
| "-1" | support path: a repeated checkIfExists from SupportForm |

For a "clean" registration without a granted phone permission, "7" is typical (live dump: a rooted device, permission not granted → 7); with the permission granted and manual input that did not match the SIM — "5".

The key invariant: the value lives in the pref until the next number entry, so in the /v2/code and /v2/register of one session it MUST match.

## Examples

- Live dump (SM-A325F, without phone permission): `mistyped=7` in both requests.
- A phone with a SIM, the user accepted the autofilled number: `mistyped=4`.
- A phone with a SIM, the number entered manually and matching the SIM (flash-eligible): `mistyped=0`.
