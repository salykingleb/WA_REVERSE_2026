# pid — OS process identifier (Process.myPid)

## Place in the flow

`pid` — the real PID of the WhatsApp process at the moment the request is built. It is filled
into the common set of registration parameters `IF6.A0O` (X/IF6.java:1139-1158), which is used by
BOTH requests: `/v2/code` (generateAuthCode) and `/v2/register` (verifySecurityCode), as well as
by standalone code requests. In the capture facts: `pid=28104` is present in code and register
**identically** — both requests went from one process without a restart between them.

## Wire format (+LIVE example from the capture)

A decimal string of an integer without leading zeros and without a sign.

LIVE example (Samsung SM-A325F, 2026-09-22):
```
pid=28104
```
— 5 digits, without any encodings (all characters `[0-9]` are unreserved, percent-encode is not
needed). Position on the wire (the native order): in code — between `rc` and `_ge`; in register —
between `network_radio_type` and `_ge`.

## How it is formed in the app

`IF6.A0O(IF6 if6, Map map)` — the common builder of environment parameters (X/IF6.java:1139-1158):
```java
public static final void A0O(IF6 if6, java.util.Map map) {
    map.put("network_radio_type", ...);
    map.put("simnum", ...);
    map.put("hasinrc", AbstractC87053t4.A1b(GCN.A1W(application.getFilesDir(), "rc2") ? "1" : "0", charset));
    map.put("pid", AbstractC87053t4.A1b(String.valueOf(Process.myPid()), charset));   // ← строка 1155
    A0Z(map);
}
```
- The source is `android.os.Process.myPid()` — the getpid() syscall of the current process. No
  generation/randomization: the value is fully determined by the OS.
- `String.valueOf(int)` → a decimal string; `AbstractC87053t4.A1b(..., charset)` — a byte-encoding
  wrapper (essentially toByteArray(charset), for digits = ASCII).
- The `A0O` calls are inlined into the assembly of the common parameter map of the registration
  flow (see, for example, the construction of a LinkedHashMap with
  mistyped/client_metrics/entered/… → `A0O(if6, map)` → `A0R(...)`), therefore pid appears in all
  the derived requests of this flow.

**Persistence:** none — it is a property of the process, not of the installation. The value
changes on every new process launch (the typical Android range: thousands to tens of thousands).
Within one launch (one registration session: code → register) — unchanged.

## Native implementation

Not involved: the value is computed in Java and passed into the map as a ready string. The name
`"pid"` is in the native table of registration parameter names (rodata libwhatsapp.so). The
process is the same for the Java and the native side, therefore pid is consistent with any native
metrics tied to pid (for example, client_metrics/uptime of one process).

## Value selection conditions (variants, ranges, when absent)

- An integer ≥ 1 (pid 0 is the system idle, unreachable for an app).
- On Android the values usually lie in the range ~1000-50000: lengths of 4-5 digits dominate,
  3-digit ones also occur (low pids after boot), and large ones (>50000) with active device
  usage.
- **Any digit can be first, and ALL digits 0-9 occur in the composition, including 0, 1 and 9** —
  the live reference `28104` contains both `0`, and `1`, and `8` (the OS hands out the first pid
  values to processes, the distribution is uniform over the actual process numbers).
- The parameter is always sent (there is no null branch); the same in all the requests of one app
  launch.

## Cryptography and encoding

None. Number → decimal string → ASCII. Percent-encoding is never required (digits are
unreserved).

## Examples

- Live: `pid=28104` (code and register, one process — indirectly confirms that both requests were
  made from one launch without a restart; live capture).
- Behavior on an app restart: pid will change (a new value from the OS), the other persistent
  identifiers (id, backup_token, fdid, expid, access_session_id) will not.
- Format: `String.valueOf(Process.myPid())` — no paddings/prefixes.
