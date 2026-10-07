# hasinrc, db — local registration state (rc2 storage) and the developer mode flag (db)

## Purpose

- **hasinrc** — "whether the device already has entries in the recovery storage": whether the directory `files/rc2` exists — WhatsApp's encrypted local storage of backup tokens. The flag "this device has already seen the registration flow" from the file system's point of view.
- **db** — **the flag of developer mode enabled on the phone** (developer options / debugging). It is collected NOT in Java code: the only source is the native parameter map from wamsys (JniBridge), the contents are obfuscated in libs.so. Live dumps give the value "1" — the reference SM-A325F (a root device for reverse engineering) ran with developer mode enabled, which is consistent.

## Place in the flow

| Parameter | /v2/code | /v2/register | /v2/exist |
|---|---|---|---|
| hasinrc | yes | yes | yes |
| db | yes | yes | yes (inherited from the shared native map) |

## Format

The strings "0"/"1":

```
hasinrc=1
db=1                ← живой пример: оба "1" в обоих запросах (SM-A325F)
```

## How it is formed in the application

### hasinrc

`X/IF6.java:1154` (inside the shared collector A0O):

```java
map.put("hasinrc", GCN.A1W(application.getFilesDir(), "rc2") ? "1" : "0");
```

`GCN.A1W(File, String)` is `new File(file, str).exists()`: a simple check of the existence of the file/directory `<filesDir>/rc2`.

What rc2 is — the encrypted storage of backup tokens (class `X/C00L.java`): `A09()` writes tokens, `A0I()` reads them. The directory is created:
- `X/C00L.java:206–214` — on the first token write;
- `RunnableC42361IqZ.java:178` — repository initialization (GCM.A1C creates rc2).

The key order of operations: before EVERY /v2/code the application calls `IF6.A0u()`/`A0t()` (obtaining/generating the backup token), which, when there is no token, generate a new one and write it via `C00L.A09` — **the rc2 directory is created BEFORE the parameter map is built**. Therefore by the moment of any real request hasinrc is almost always "1".

### db

In the Java code of /v2/code and /v2/register the parameter `"db"` is **not collected at all** (grep over the jadx tree yields only protobuf/document classes). The only way additional fields appear in the maps of both requests is the native map:

`X/IF6.java:553–567`, method `A0S()`:

```java
JniBridge.jvidispatchIOO(7, application, jniBridge.getWajContext());   // инициализация wamsys
Map map2 = (Map) JniBridge.jvidispatchOOO(16, application2, jniBridge2.getWajContext());
map.putAll(map2);   // native-поля вливаются в общую карту запроса
```

`com/whatsapp/wamsys/JniBridge.java` — a generic dispatcher into the native library; the parameter name strings are absent in the .so (verified by a scan of the library), the map contents are obfuscated. Live dumps show that the final set of native fields includes `db=1` (stably "1" in code and register).

**Semantics: `db` = "developer mode is enabled on the phone"** (developer options / debugging).
The native side detects the developer mode state of the device and sends it to the server as an
anti-fraud signal: enabled debugging is a typical sign of a device being researched / prepared
for manipulation. The exact native marker (which particular source is read —
`Settings.Global.DEVELOPMENT_SETTINGS_ENABLED`, `ADB_ENABLED`, or usb properties of the system) was not
established in the obfuscated libs.so; from the outside only the final "0"/"1" is visible.

## Value selection conditions

### hasinrc

| Value | Condition |
|---|---|
| **"1"** | the directory `files/rc2` exists. The norm: the backup token is generated and written before the request is built; after the first code request — always "1" |
| **"0"** | the directory has not been created yet — practically unattainable by the moment of a real /v2/code (only if the token write failed) |

### db

| Value | Condition |
|---|---|
| **"1"** | developer mode on the device ENABLED (live dump: SM-A325F with developer mode/root enabled — stably "1" in both requests) |
| **"0"** | developer mode disabled (not observed in live captures — both reference snapshots were taken on a device with debugging enabled) |

## Examples

- Live dump (SM-A325F, fresh registration session): `hasinrc=1`, `db=1` — identical in /v2/code and /v2/register.
- The first code request after installation: hasinrc is already "1" (the token was generated and written before the map was built).
