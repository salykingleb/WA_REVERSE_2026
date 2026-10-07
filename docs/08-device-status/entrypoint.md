# entrypoint — where registration was launched from ("suma" | "create_paa" | absent)

## Purpose

A label of the registration flow launch source. Tells the server from which place of the application the request was initiated: the standard number sign-in flow (suma — SUMA = number entry on a device with account history) or the PAA flow (Primary Account Add — adding a primary account via the PMA overflow menu). May be absent — on devices without account history.

## Place in the flow

| Request | Presence |
|---|---|
| /v2/code | yes (with a counter > 0 or PAA) |
| /v2/register | yes (the same condition) |
| /v2/exist | yes (the same collector A0T) |

Placed by the shared method `IF6.A0T()` (`X/IF6.java:569–578`), which is called from the map builders of ALL requests — the value is computed from a single persistent state, so in the code and register of one session it is always consistent.

## Format

A literal string:

```
entrypoint=suma        ← живой пример: "suma" в ОБОИХ запросах (SM-A325F)
```

## How it is formed in the application

`X/IF6.java:569–578`:

```java
public static final void A0T(IF6 if6, java.util.Map map) {
    if (((SharedPreferencesOnSharedPreferenceChangeListenerC04950Me) ...).A0D()) {
        map.put("entrypoint", B0W.A1Z("create_paa"));      // PAA-флоу включён
    }
    if (...A0D() || A02(if6).A0C().A03() <= 0) {
        return;                                            // PAA или счётчик 0 → без suma
    }
    map.put("entrypoint", B0W.A1Z("suma"));
}
```

Two inputs:

1. **The PAA flag**: `C04950Me.A0D()` — the pref `paa_from_pma_in_overflow_menu` (the flow of adding a primary account via the PMA overflow menu). If enabled — `"create_paa"` is set and the method returns immediately (suma is unreachable).
2. **The inactive accounts counter**: `A02(if6)` → `C018208l` (WaSharedPreferencesFacade) → `.A0C()` → the class `C1DB` (`X/C1DB.java`): `A03() = prefs.getInt("number_of_inactive_accounts", 0)`. This is the number of NOT active accounts known to the device's account-switching repository. If > 0 → `"suma"`; if 0 → the key is not added at all.

### When the counter is written — BEFORE the first code request

- **RegisterPhone (the number entry screen)** — `RunnableC42361IqZ.../RunnableC42361Iqc.java:111` (case 11), posted during the screen UI initialization (`RegisterPhone.java:2957/2980/3005`), i.e. BEFORE pressing "Next" and BEFORE /v2/code: `A04(accountSwitchingRepo.A07().size())`.
- `GEQ` = AccountSwitchingDataRepo: `A07()` — all repository accounts EXCEPT the current one (including logged-out ones). The repository lives in the file `<data>/app_account_switching/accounts`; it survives updates and logout, and is erased on uninstall/wipe.
- Other writers: `GD2.BYr` (AccountSwitchingAsyncInit, application start), `Main.java:321`, `A4K` (RemoveAccountUseCase), `C40732I1j` (AddAccountNavigator); zeroing: HomeActivity with an empty repository (`C9V5` case 5), `C38452Gyd:58`.
- Repository replenishment: LogoutManager (`J6S`/markCurrentAccountLoggedOut) writes the current account as logged-out — including on FORCED_REGISTRATION (displacement by registering the number on another phone).

Consequence: there is NO increment "after the first request" — the state is ready before number entry. A phone with history (logout, second number, displacement) yields suma already in the FIRST request of the session (the live dump — exactly so).

## Value selection conditions

| Value | Condition | Probability/frequency |
|---|---|---|
| **"suma"** | the PAA pref is off AND `number_of_inactive_accounts > 0` (the device has account history: logout/displacement/second number) | the dominant share of live re-registration traffic |
| **"create_paa"** | the pref `paa_from_pma_in_overflow_menu` enabled (the PAA flow via the PMA overflow menu) | a rarity; unreachable for regular registration |
| (absent) | PAA off AND the counter == 0 — a clean installation without WhatsApp history on the device (the first registration after installation; reinstall/wipe; HomeActivity zeroed it) | the "first sign-in after installation" marker |

Consistency: the value is the same in /v2/code, /v2/register, and /v2/exist (one collector over persistent state).

## Examples

- Live dump (SM-A325F with account history): `entrypoint=suma` in both requests, including the very first code request of the session.
- A freshly installed WhatsApp on a clean device: the key is absent in both requests.
- Adding a primary account via the PMA menu: `entrypoint=create_paa`.
