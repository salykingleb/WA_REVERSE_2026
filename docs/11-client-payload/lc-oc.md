# Field 24 lc (successful login counter) and field 23 oc (signature check)

ClientPayload `X/C1HS.java`, build 2.26.35.75 (263507522). The two fields are set by
the fillers `X.1J8` (lc) and `X.1Hs` (oc) — DI cases 6559 and 6556 in `X/C1US.java`.

## lc (field 24, int32) — login count

Constant: `LC_FIELD_NUMBER = 24` (`X/C1HS.java:34`). The getter `C1HS.A00()` returns
exactly `lc_` — which is why this field is printed in the
`handshake_payload login_count=` log.

Writing into the payload (`X/C1US.java:20352-20356`):

```java
int i8 = ((C018208l) this.A02.A00.get()).A0I().A02()
        .getInt("connection_lc", 0);
c1hs7.bitField0_ |= 131072;
c1hs7.lc_ = i8;
```

The counter lives in the SharedPreferences store `C1FO` (original `X.1FO`, successor
of `C0FF`):

- increment after a successful login — `C1FO.A04()` (`X/C1FO.java:19-26`):
  ```java
  int i = A02().getInt("connection_lc", 0);
  int i2 = i + 1;
  if (i == Integer.MAX_VALUE) i2 = 0;   // защита от переполнения — wrap в 0
  A01().putInt("connection_lc", i2).apply();
  ```
- reset to 0 — on registration/key re-creation: `AnonymousClass139`, after writing a
  new static keypair, executes `putInt("connection_lc", 0).apply()`
  (`X/AnonymousClass139.java:352`); before the increment the default value is also 0
  (`AbstractC49522Fq.A1N(prefs, "connection_lc", 0)` in `X/C41722Ifn.java`).

Properties:

- Always present in the payload (unconditional write). On **first login** — always
  `0`: the increment happens only after a parsed `<success>`, and registration has
  just zeroed the pref.
- The meaning for the server is "how many successful logins this installation has
  already had with the current keys": a small lc + an unfamiliar set of fields — a
  fresh installation; a large lc with a sharply changed userAgent — grounds for a
  device-change heuristic.
- Next to them, the same `C1FO` prefs store `connection_sequence_attempts` (`A05`)
  and `last_successful_connection_step/host/port` (`A06`) — these are inputs for
  `connectionSequenceInfo` (field 43), not for lc.

In the first-login capture this is the account's first chat login (creation and t
differ by 6 seconds) — in this session's payload `lc = 0`.

## oc (field 23, bool) — two-stage APK signature check

Constant: `OC_FIELD_NUMBER = 23` (`X/C1HS.java:37`). The value is computed by the
`X.1Hs` filler in two steps (`X/C1US.java:19971-20009`).

### Stage 1: C1Iv.A00 (version range + SHA-1 in base64url)

```java
Application application = this.A02;              // C00I.A00()
boolean z = C1Iv.A00(application) == 1;
c1hs2.bitField0_ |= 65536;
c1hs2.oc_ = z;                                   // временно oc = результат ступени 1
```

`X/C1Iv.java:16-47` (`C1Iv.A00`, the result is cached in the static `Long A00`):

```java
long jA00 = C1Iw.A00(context, context.getPackageName());   // versionCode (long)
if (jA00 < 263507522 || jA00 > 263507530) return 0;        // узкий диапазон сборки
Signature[] sigs = C1Iw.A07(context, packageName);         // GET_SIGNATURES/GET_SIGNING_CERTIFICATES
MessageDigest md = MessageDigest.getInstance("SHA-1");
md.update(sigs[0].toByteArray());
String s = Base64.encodeToString(md.digest(), 11);         // флаг 11 = URL_SAFE|NO_PADDING|NO_WRAP
return "OKD31QX-GP7GT780Psqq8xDb15k".equals(s) ? 1 : 0;
```

The conditions for returning 1 — all at once:

1. the package's `versionCode` lies in the range **[263507522, 263507530]** — i.e.
   exactly this 2.26.35.x line (the current build 263507522 is the lower bound);
2. the SHA-1 of the APK's first signing certificate in base64url without padding
   equals `OKD31QX-GP7GT780Psqq8xDb15k`.

This is the same certificate as the official `com.whatsapp`: in the HTTP capture
(live capture, decrypted gpia blob) it is visible as standard base64
`cert="OKD31QX+GP7GT780Psqq8xDb15k="` — converting to URL-safe without padding
(`+`→`-`, `/`→`_`, `=` dropped) yields exactly the constant from `C1Iv`.

If the versionCode is out of range or the SHA-1 did not match — `C1Iv.A00` returns 0,
`oc=false` is written immediately, and the second stage is not executed.

### Stage 2: C40715I0p.A00 overwrites oc

Executed **only if oc=true after stage 1** (`X/C1US.java:19987-20008`):

```java
if (c1hs4.oc_) {
    if (this.A00 == null) {
        this.A01 = application.getPackageName();
        this.A00 = IBW.A00(packageManager, this.A01);   // строгая выборка подписи
    }
    C40715I0p c40715I0p = (C40715I0p) this.A09.A00.get();
    boolean z2 = !c40715I0p.A00(this.A01, this.A00.toByteArray());
    c1hs5.oc_ = z2;                                     // oc = НЕ("проверка пройдена")
}
```

`IBW.A00` (`X/IBW.java`) is a strict version of the extraction: it verifies the
package name, requires exactly one signature (`"Multiple signatures not supported"`),
and throws exceptions on mismatch instead of returning null.

`X/C40715I0p.java:A00(packageName, signatureBytes)` is a second, independent
verification against obfuscated constants (the result is cached in the `Boolean A00`
field):

- first, a transformation of the **package name bytes** via `B0X.A1V` and XOR with
  the 5-byte key `A01 = {-64,-64,-84,?,-27}`, comparison against the constants `A02`
  (16 bytes) / `A03` (12 bytes) — a package name whitelist;
- then `GCK.A0z().digest(signature)` — the **SHA-256** of the signature, the same XOR
  transformation, comparison against the 32-byte constants `A04`/`A05`.

Any exception of the whole stage 2 (PackageManager unavailable, signature unreadable)
is caught by `catch (Exception e) { Log.e(e); }` — and `oc` **remains true**.

### Resulting semantics

| Situation | oc on the wire |
|---|---|
| A genuine Play installation (version in range, SHA-1 matched, second verification passed) | **false** |
| versionCode outside [263507522, 263507530] | false (both stages miss) |
| SHA-1 mismatched at stage 1 (a re-signed APK, same version) | false — stage 2 is not executed |
| Stage 1 passed, but stage 2 (package name / SHA-256 against obfuscated constants) did not match | **true** |
| Stage 1 passed, stage 2 threw an exception | **true** (the stage 1 value remains) |

Decoding the name: `oc = false` is the norm. `oc = true` means "the client fell
within the official version range and with the official certificate SHA-1, but the
additional verification (SHA-256 path / package name against the baked-in constants)
did not match or could not run" — the server marks such an installation as
suspicious. For an emulator reverse-installation with someone else's signature, the
typical case is precisely "SHA-1 mismatched at stage 1" → oc=false, but then other
checks (`_gi`, attestations after `<success>`) will expose the re-signing regardless
of oc.

Both values of the field are written with bit `65536` (`bitField0_`); the field is
always present in the payload, set by the `X.1Hs` filler.

## Where to look

- lc: `X/C1US.java:20352-20356`; store `X/C1FO.java:19-30`; reset
  `X/AnonymousClass139.java:352`; the `login_count=` log — `X/C0SW.java:1645`.
- oc: stage 1 — `X/C1Iv.java` (range + SHA-1 base64url), call at
  `X/C1US.java:19971-19976`; stage 2 — `X/C1US.java:19987-20008`, verification
  `X/C40715I0p.java`, signature extraction `X/IBW.java`, `X/C1Iw.java` (`A00`
  versionCode, `A07` signatures).
- Live certificate: the decrypted gpia blob (`cert`, `_icr`).
