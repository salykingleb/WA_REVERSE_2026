# has_play_store — indicator of an installed Google Play Store

## Position in the flow

A POST /v2/register parameter from the additional-parameters core (IF6.A0I):
tells the server whether the device has the Google Play store (the
`com.android.vending` package). It is an environment signal: devices with
Play Store almost certainly have GMS, receive an advertising ID, Play
Integrity (gpia), and Firebase identifiers; the absence of Play Store is a
marker of forked firmware (Huawei/HarmonyOS without GMS, AOSP builds,
emulators), for which the server expects a different profile of the other
parameters (for example, the absence of gpia/fid/advertising_id).

## Format (+live example)

The string `"true"` or `"false"`.

Live example (dump 2026-09-22, Samsung SM-A325F, stock firmware with GMS):
`has_play_store=true`.

## How it is formed in the app (class file:line, code)

```java
// X/IF6.java:381 (inside A0I)
linkedHashMapA0O.put("has_play_store",
    AbstractC87053t4.A1b(String.valueOf(
        AbstractC49532Fr.A1Y(                       // A1Y(obj) = (obj != null)
            C1Iw.A02(if6.A03, "com.android.vending"))),
        C07k.A05));
```

`C1Iw.A02` (X/C1Iw.java:76–90, PackageManagerUtils) — a safe wrapper over
PackageManager:

```java
public static PackageInfo A02(Context context, String str) {
    try {
        PackageManager packageManager = context.getPackageManager();
        if (packageManager != null) {
            return packageManager.getPackageInfo(str, 0);   // flags = 0
        }
        return null;
    } catch (PackageManager.NameNotFoundException unused) {
        Log.i("PackageManagerUtils/Failed to get package info for:" + str);
        return null;
    } catch (RuntimeException e) {                          // "Package manager has died"
        Log.e("Package manager has died", e);
        return null;
    }
}
```

In total: `has_play_store = "true"` ⇔ `PackageManager.getPackageInfo(
"com.android.vending", 0)` returned a PackageInfo (the package is visible to
the system), otherwise `"false"`. Important semantic details:

- it is **package visibility** that is checked, not the enabled state or the
  version (ApplicationInfo.enabled/version are checked by other C1Iw methods —
  A0A/A0B — but they are not used here);
- on Android 11+ (API 30), visibility of third-party packages is limited by
  `<queries>`: the WhatsApp manifest includes a visibility request for
  com.android.vending, so on GMS devices the check works normally;
- exceptions (package not found / package manager died) → null → `"false"`.

## Value selection conditions

- **true** — on any device with Google Play services (ordinary Android with
  GMS). The default value for the live flow.
- **false** — when the com.android.vending package is not visible:
  - Huawei/Honor without GMS (AppGallery instead of Play);
  - clean AOSP/LineageOS builds without GApps;
  - some emulators and virtual containers;
  - degenerate cases of package manager crashes (RuntimeException).
- The field is always present in the request (an unconditional put in A0I);
  only the value changes.
- Consistency with the other parameters: with has_play_store=false, the
  server does not expect advertising_id (Google Play Services), fid (Firebase
  Installations), or a live gpia from the same device — their
  absence/presence must match this flag.

## Cryptography and encoding

- No cryptography: boolean → "true"/"false" → UTF-8 bytes → percent-encoding
  (letters do not change).

## Examples

- GMS smartphone (live dump): `has_play_store=true`.
- Huawei without GMS: `has_play_store=false` (+ advertising_id, fid, and the
  gpia token are absent).
