# Fields 36 yearClass and 37 memClass — device performance classes

ClientPayload `X/C1HS.java`, build 2.26.35.75 (263507522). Both fields are set by
the `X.1Hs` filler (DI case 6556) right after userAgent and oc
(`X/C1US.java:19977-19986`):

```java
int iA01 = C1Ix.A01((C00R) this.A07.A00.get(), (C0AO) interfaceC001600s.get());
c1hs3.bitField0_ |= DexConstants.FB4A_LINEAR_ALLOC_BUFFER_SIZE;   // бит 36
c1hs3.yearClass_ = iA01;

int iA02 = C1Iy.A01((C0AO) interfaceC001600s.get());
c1hs4.bitField0_ |= EditorInfoCompat.IME_FLAG_NO_PERSONALIZED_LEARNING;  // бит 37
c1hs4.memClass_ = iA02;
```

Constants: `YEAR_CLASS_FIELD_NUMBER = 36`, `MEM_CLASS_FIELD_NUMBER = 37`
(`X/C1HS.java:52, 36`). Both values are computed once per installation lifetime and
cached — on every login they are simply read.

## yearClass (field 36, int32) — "performance year"

Implementation `X/C1Ix.java`. What goes into the payload is the **`A01`** result —
the "2016 scale" (extended, values 2009..2016), cache — `startup_prefs` →
`year_class_cached_value_2016_pref`. The outdated `A00` scale (cache
`year_class_cached_value_pref`) with values 2008..2014 is not used in this payload
but remains in the code and is called as a fallback.

The inputs are the helpers `X/AbstractC40471qk.java` ("deviceinfo"):

- `A00()` — maximum CPU frequency, kHz (reading
  `/sys/devices/system/cpu/cpuN/cpufreq`; the body did not decompile in normal
  jadx mode);
- `A01()` — number of cores: counting `cpu[0-9]+` directories in
  `/sys/devices/system/cpu/` (`AbstractC40471qk.java:76-87`), −1 if unavailable;
- `A02(c0ao)` — total memory: `ActivityManager.MemoryInfo.totalMem`
  (`AbstractC40471qk.java:90-104`), −1 on `am=null` or NPE.

### The A01 scale (in the payload), `X/C1Ix.java:33-77`

If RAM could not be determined (`A02 == -1`) — the median of the old `A02` scale is
taken (below). Otherwise, classification by totalMem with a CPU adjustment:

| totalMem | CPU condition | yearClass |
|---|---|---|
| ≤ 805 306 368 (768 MiB) | cores ≤ 1 | 2009 |
| ≤ 805 306 368 (768 MiB) | cores ≥ 2 | 2010 |
| ≤ 1 073 741 824 (1 GiB) | maxCpu < 1 300 000 kHz (1.3 GHz) | 2011 |
| ≤ 1 073 741 824 (1 GiB) | otherwise | 2012 |
| ≤ 1 610 612 736 (1.5 GiB) | maxCpu ≥ 1 800 000 kHz (1.8 GHz) | 2013 |
| ≤ 1 610 612 736 (1.5 GiB) | otherwise | 2012 |
| ≤ 2 147 483 648 (2 GiB) | — | 2013 |
| ≤ 3 221 225 472 (3 GiB) | — | 2014 |
| ≤ 5 368 709 120 (5 GiB) | — | 2015 |
| > 5 368 709 120 | — | 2016 |

The constant `Voip.MAX_DATA_USAGE_IN_A_CALL = 2147483648L` (2 GiB,
`com/whatsapp/calling/voipcalling/Voip.java:52`) is used in these thresholds as the
"2 GiB threshold".

### Fallback A02 (old scale), `X/C1Ix.java:79-145`

Collects up to three "years" and takes the median (an even number of elements — the
mean of the two middle ones; an empty list → −1):

| Component | Ranges |
|---|---|
| Core count (`A01`) | 1 → 2008; 2..3 → 2011; ≥4 → 2012 |
| maxCpu, kHz (`A00`) | ≤528 000→2008; ≤620 000→2009; ≤1 020 000→2010; ≤1 220 000→2011; ≤1 520 000→2012; ≤2 020 000→2013; above→2014 |
| RAM, bytes (`A02`) | ≤201 326 592 (192 MiB)→2008; ≤304 087 040→2009; ≤536 870 912 (512 MiB)→2010; ≤1 073 741 824→2011; ≤1 610 612 736→2012; ≤2 147 483 648→2013; above→2014 |

### Live calculation for the phone from the capture (SM-A325F)

The live capture records `device_ram=3,56` (GB; this is exactly
`C1Iy.A00(context, true)` = `totalMem / 1.073741824E9`). In bytes ≈ 3.82 billion —
the "> 3 GiB and ≤ 5 GiB" range → **yearClass = 2015**. The value is cached in
`startup_prefs` and goes in every ClientPayload of this installation.

## memClass (field 37, int32) — memory heap class

Implementation `X/C1Iy.A01` (`X/C1Iy.java:31-45`):

```java
ActivityManager am = c0ao.A03();                    // getSystemService("activity")
if (am == null) {
    Log.w("MemoryClassProvider/calculateHeapClass/am=null");
    return 16;                                      // дефолт при недоступном AM
}
int memoryClass = am.getMemoryClass();              // MiB per-application heap
A00 = memoryClass;                                  // static-кэш
return memoryClass;
```

Properties:

- The source is `ActivityManager.getMemoryClass()`: the per-application heap limit
  in MiB assigned by the platform (it depends on total RAM and `largeHeap`, but the
  base class is what is taken). The value is an integral number of megabytes.
- Typical values on Android: 16 (old/cheap devices), 32, 48, 64, 96, 128, 192, 256,
  512 (flagship tablets). On the SM-A325F (4 GB) usually 192 or 256 — the exact
  value is determined by the device platform and is visible only via a live
  `dumpsys meminfo` query/log.
- Cached in the static `C1Iy.A00` — repeated logins within the same session do not
  recompute the value.
- Neighboring methods of the same class do not go into the payload but are used in
  other subsystems: `C1Iy.A00(context, z)` — total/availMem in GB (double; the live
  3.56 on the captured phone, `X/C1Iy.java:14-29`), `C1Iy.A02()` — the "weak
  device" flag: `Build.MODEL` is in the blacklist `{"GT-N7100", "GT-I9305"}` or
  memClass ≤ 48 (`X/C1Iy.java:47-56`).

## Why the server needs both fields

yearClass and memClass are a stationary characteristic of the hardware: they must
match from login to login for a given installation. A jump in either of them
(2015 → 2009, 256 → 64) without a change of fdid/phoneId is a marker of moving to
another device or an environment substitution; combined with `deviceType`,
`deviceModelType`, and `phoneId` (userAgent field 9) this is a cheap
fingerprint-style integrity check of the environment.

## Where to look

- Call site in the payload: `X/C1US.java:19977-19986` (X.1Hs).
- yearClass: `X/C1Ix.java` (`A01` — the scale used in the payload, `A00` — the
  outdated one, `A02` — the median fallback scale).
- CPU/RAM helpers: `X/AbstractC40471qk.java` (`A00` frequency, `A01` cores, `A02`
  totalMem).
- memClass: `X/C1Iy.java` (`A01` — getMemoryClass, `A00` — GB, `A02` — low-mem).
- The live `device_ram=3,56` GB (registration capture).
