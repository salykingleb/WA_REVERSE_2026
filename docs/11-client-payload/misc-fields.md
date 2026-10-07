# Other ClientPayload fields: 41, 40, 45/46, 47, 18

ClientPayload `X/C1HS.java`, build 2.26.35.75 (263507522). Collected here are the
fields that are absent on a "regular" phone first login or are almost always the
same, but become significant in special states of the installation.

## lidDbMigrated (field 41, bool) — LID database migration

Constant: `LID_DB_MIGRATED_FIELD_NUMBER = 41` (`X/C1HS.java:35`).

The `X.1J8` filler code (`X/C1US.java:20307-20317`):

```java
if (c1hm.A04 != null && !c1hm.A0A) {              // jid есть И signal-store не новый
    if (((C03580Gs) this.A00.A00.get()).A00 != null) {   // состояние миграции инициализировано
        boolean zA01 = ((C03580Gs) ...).A01();    // isGlobalChatDbMigrated
        c1hs.bitField0_ |= 134217728;
        c1hs.lidDbMigrated_ = zA01;
    }
}
```

The condition was verified against two independent decompilations — 2.26.35
(`X/C1US.java:20307`) and 2.26.37 (`X/C1UE.java:21233`,
`if (c1e0.A04 != null && !c1e0.A0B)`): both give "JID present AND
`signal_protocol_store_is_new` == false". Thus the field is set on **repeat phone
logins** with an old signal store; it is not set on the first login (the store is
still marked new) nor for a companion (no JID). In the early-analysis table the
condition was recorded the other way around ("no user JID") — the code is what is
correct: `C1HM.A04 != null && !C1HM.A0A`.

The flag itself is `X/C03580Gs.java` (ChatLidMigrationState): `A00` is the cached
`Boolean isGlobalChatDbMigrated`; `A01()` (`X/C03580Gs.java:68-79`) returns the
cache, and in an uninitialized state writes the metric
`ChatLidMigrationState/isGlobalChatDbMigrated "msgStore not ready"` and returns
`true`. Through the filler this default is unreachable: the `A00 != null` guard
filters out the uninitialized state. The computation is based on whether the global
chat-DB migration has completed (pref
`global_chat_db_migration_completed_on_primary`, companion accounting via
`C08X.BKV()`).

## trafficAnonymization (field 40, enum C1J0)

Constant: `TRAFFIC_ANONYMIZATION_FIELD_NUMBER = 40`. The enum `X/C1J0.java`:
`OFF = 0`, `STANDARD = 1` — there are no other values.

The `X.1Iz` filler (DI case 6557, `X/C1US.java:20012-20026`) — unconditionally,
always one of the two:

```java
C1J0 c1j0 = ((C10270df) this.A00.A00.get()).A0L() ? C1J0.STANDARD : C1J0.OFF;
c1hs.trafficAnonymization_ = c1j0.getNumber();
```

`X/C10270df.java:20-26` (`A0L`):

```java
if (this.A00.A0w(9370)) {                       // AB-флаг 9370 включён
    return this.A01.A02(C02S.A00)               // per-user pref "traffic anonymization"
        || this.A02.A05();                      // серверный конфиг
}
return false;
```

So `STANDARD` = AB experiment 9370 is active AND (the user enabled traffic
anonymization in their settings OR it is prescribed by the server). In all other
cases — `OFF` (0). The user toggle is a per-user pref (`X/C10280dg.java:11-14`, the
key is derived from the user number `C02S.A00`), the server part is
`X/C10290dh.java`.

## processingQueueSize (46) and preacksCount (45) — queue snapshot

Constants: `PROCESSING_QUEUE_SIZE_FIELD_NUMBER = 46` (written with bit 1 of
`bitField1_`), `PREACKS_COUNT_FIELD_NUMBER = 45` (the `Integer.MIN_VALUE` bit of the
upper word).

The source is `C1HM.A07`, an `X/C71163Gf.java` object ("ClientQueueState"):

```java
public String toString() {
    return "ClientQueueState(preacksCount=" + A00 + ", processingQueueSize=" + A01 + ")";
}
```

- `A00` → field 45 `preacksCount` — the number of preacks: the queue of "sent but
  not yet acknowledged" (`C1E5.A0B()` = `A01 + A0G.size()`, `X/C1E5.java:481-489`);
- `A01` → field 46 `processingQueueSize` — the total size of the outgoing queue
  (`C1F2.A04()` = the sum of three pending counters, `X/C1F2.java:92-96`).

The `X.1J8` filler (`X/C1US.java:20324-20340`):

```java
C71163Gf c71163Gf = c1hm.A07;
if (c71163Gf != null) {
    Log.i("SessionInfoProvider clientQueueState=" + c71163Gf);
    c1hs3.processingQueueSize_ = c71163Gf.A01;
    c1hs4.preacksCount_ = c71163Gf.A00;
}
```

The snapshot is created in `C0SW.A1S(passive)` (`X/C0SW.java:738-744`) BEFORE the
payload is assembled:

```java
if (c016407s.A0w(20485) || (z && c016407s.A0w(22413))) {   // AB 20485 всегда, либо AB 22413 && passive
    return new C71163Gf(((C1E5) this.A0b.get()).A0B(), ((C1F2) this.A0a.get()).A04());
}
return null;   // поля 45/46 в пейлоад не попадают
```

Consequence: the fields are absent when both AB flags are off, and on any login
without a queue if 22413 requires passive. On the phone's first login there is no
queue; in the first-login capture the FunXmpp hook does not see the payload, but the
absence of the fields is indirectly visible from the fact that right after
`<success>` a `<presence type='available'/>` goes out without passive signs.

## pairedPeripherals (field 47, repeated string)

Constant: `PAIRED_PERIPHERALS_FIELD_NUMBER = 47`. The only payload field with
strings (not an enum): the possible elements are `smart_glasses` and `garmin`.

The `X.1JC` filler (DI case 6567, `X/C1US.java:20745-20789`), chain of conditions:

1. `C08X.Aoo()` — the account's PhoneUserJid — is not null;
2. the peripherals provider's `Optional` (DI 7375, `C1JD`) is present;
3. the AB experiment `C1JE.A00 = new C09O(27619, false, true)` is enabled
   (`X/C1JE.java:6`, the check `c00d.A0z(c09o)`).

Then the set is collected:

```java
for (C29015Cnk cnk : ((C06060Qs) c1jd.A00.A00.get()).A0P()) {
    if (areEqual(cnk.A0A.userJid, phoneUserJid) && cnk.A0B.ordinal() == 24) {
        linkedHashSet.add("smart_glasses");       // умные очки, привязанные к этому номеру
    }
}
if (c1jd.A01.isPresent() && !((C1JR) c1jd.A01.get()).A0L().isEmpty()) {
    linkedHashSet.add("garmin");                  // привязанные часы Garmin
}
if (listA1F.isEmpty()) return;                    // пусто — поле 47 не ставится
c1hs.pairedPeripherals_.addAll(list);
```

`ordinal() == 24` is the "glasses" category of a linked device in the internal
companion enum (`C29015Cnk.A0B`); the link is checked against the phone JID, so
another number's peripherals will not get into the payload. A regular phone without
peripherals or without AB 27619 — the field is absent entirely.

## device (field 18, int32) — companion device id

Constant: `DEVICE_FIELD_NUMBER = 18`. The `X.1JU` filler (DI case 6560,
`X/C1US.java:20407-20414`):

```java
if (((C017908i) this.A03.A00.get()).BKW(false)) {
    int i2 = ((C018308m) this.A02.A00.get()).A01.A00.getInt("registration_device_id", 0);
    c1hs.bitField0_ |= 2048;
    c1hs.device_ = i2;
}
```

`C017908i.BKW(false)` (`X/C017908i.java:814-825`) is true only when the pref
`registration_device_id > 0`, i.e. this installation is a companion (linked device)
and has already been assigned a server device number. On the "owner" phone the pref
is empty → `BKW` false → field 18 is absent from the payload. The value is an int
assigned when the companion was linked; together with `username` (absent for a
companion) and `devicePairingData` (19) it forms the companion's identity.

## Presence summary table

| Field | Phone first login | Phone repeat login | Companion |
|---|---|---|---|
| 41 lidDbMigrated | no (store is new) | yes, if the migration is initialized | no (jid null) |
| 40 trafficAnonymization | OFF (or STANDARD with AB+setting) | same | same |
| 45/46 queue | no (no snapshot/queue) | with AB 20485 — always with the queue | same |
| 47 pairedPeripherals | no | only with AB 27619 and linked peripherals | same |
| 18 device | no | no | yes (registration_device_id) |

## Where to look

- lidDbMigrated: `X/C1US.java:20307-20317`, `X/C03580Gs.java`; cross-check
  `X/C1UE.java:21233` (2.26.37 decompilation).
- trafficAnonymization: `X/C1US.java:20012-20026`, `X/C10270df.java`, `X/C10280dg.java`.
- Queue: `X/C0SW.java:738-744` (`A1S`), `X/C71163Gf.java`, `X/C1E5.java:481`,
  `X/C1F2.java:92`.
- Peripherals: `X/C1US.java:20745-20789`, `X/C1JE.java`, `X/C1JD.java`, `X/C1JR.java`.
- device: `X/C1US.java:20407-20414`, `X/C017908i.java:814-825`.
