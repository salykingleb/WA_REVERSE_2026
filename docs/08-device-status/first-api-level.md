# first_api_level — the API level the device shipped with

New registration parameter in 2.26.44.90 (it is absent from the 2.26.35.75 parameter set).

## Place in the flow

Sent in the `POST /v2/code` and `POST /v2/register` parameter maps, next to the other
device parameters (`device_ram`, `db`, `hasinrc`, `mistyped`, …). It is a plain decimal
string, not encrypted.

## Wire format

```
first_api_level=30
```

- Decimal integer, no quotes, no padding.
- In the 2.26.44.90 capture the value is `30` (Samsung SM-A325F, Android 13, shipped with
  API 30).

## How the app builds it

`X/AbstractC43828JXg.java`:

```java
public static final Integer A00() {
    Number number;
    int intValue = (!C06z.A06 || (number = (Number) C06z.A01(
            C06z.A01, AbstractC83013lP.A1a("ro.product.first_api_level", 0, 0))) == null)
            ? 0 : number.intValue();
    Integer valueOf = Integer.valueOf(intValue);
    if (intValue <= 0) {
        return null;
    }
    return valueOf;
}
```

- Source: the system property **`ro.product.first_api_level`** (the API level the device
  was launched with, i.e. `Build.VERSION.FIRST_SDK_INT`).
- If the property is missing or `<= 0`, the method returns `null`.

The value is inserted into the request map by `X/K8v.A0R`:

```java
if (!((C0DA) C010605a.A03(k8v.A05)).A14(39257)
        || (A00 = AbstractC43828JXg.A00()) == null) {
    return;
}
map.put("first_api_level", AbstractC82993lN.A1a(AbstractC25157B0f.A1H(A00)));
```

- Gate: **AB flag 39257** (`A14(39257)`), so the parameter appears only when the AB prop is
  enabled for the account/device.
- If the property is absent, the key is **not added at all** (no empty value).

## Value selection conditions

| Condition | Result |
|---|---|
| `ro.product.first_api_level` > 0 and AB 39257 enabled | the value is sent |
| property missing / 0 | parameter absent |
| AB 39257 disabled | parameter absent |

The value is constant for a device model (it does not change with OS upgrades): a phone
that shipped with Android 11 reports `30` even after upgrading to Android 13.

## Relation to other parameters

- `android_version` (in `userAgent` of the login payload, `os` field) is the **current**
  OS version; `first_api_level` is the **launch** API level and is always `<= android_version`.
- For a device generator: take `first_api_level` from the device database
  (`devices.first_api_level`), falling back to the current `android_version` when unknown.

## Examples

- SM-A325F (Android 13, launched with Android 11): `30`.
- Pixel 7 (Android 13, launched with Android 13): `33`.
