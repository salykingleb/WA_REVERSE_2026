# Fields 1 username and 3 passive

ClientPayload `X/C1HS.java`, build 2.26.35.75 (263507522). The two fields are set by
the fillers `X.1J8` and `X.1JV` from the DI factory `X/C1US.java`; below are the
exact conditions and live values.

## username (field 1, uint64)

Constant: `USERNAME_FIELD_NUMBER = 1` (`X/C1HS.java:53`). Set by the case 6561 filler
(`X.1JV`, `X/C1US.java:20686-20734`):

```java
public void AAR(C1HM c1hm, C1Ho c1Ho) {
    UserJid userJid = c1hm.A04;                       // jid из XmppLoginContext
    if (userJid != null) {
        boolean zA0b = AbstractC02830Cv.A0b(userJid);  // jid — LID?
        String strA03 = zA0b ? userJid.user : C1Hp.A03(userJid);
        long j = Long.parseLong(strA03);               // не число → AssertionError
        c1hs.username_ = j;                            // бит 1 в bitField0_
        if (zA0b) {
            Log.i("IdentityInfoProvider using lid for login");
            c1hs2.product_ = C1JY.WHATSAPP_LID.getNumber();  // поле 20 = 4
        }
    }
    ...
}
```

Rules:

- The source is the account user JID (`C1HM.A04`). Login always takes the **user**
  part of the JID, not the server part: for `6615@s.whatsapp.net`, `6615` goes into
  the field.
- If the JID is an LID (`AbstractC02830Cv.A0b`), `userJid.user` is taken (the number
  after `@lid`), the log line `IdentityInfoProvider using lid for login` is written,
  and the `product` field (20) = `WHATSAPP_LID` (4, enum `C1JY`: WHATSAPP 0,
  MESSENGER 1, INTEROP 2, INTEROP_MSGR 3, WHATSAPP_LID 4) is additionally set.
- For a regular `pn` JID, `C1Hp.A03(userJid)` is used — numeric prefix extraction.
- The prefix must parse into a long: otherwise `AssertionError "jid prefix not numeric"`
  — the connection fails before sending.
- **No JID → the field is not set at all.** `c1hm.A04 == null` means companion
  (linked device) registration: a companion has no number of its own, and the server
  identifies it by `devicePairingData` (field 19) and `device` (18). The branch is
  identical to the check in `C0SW.A0v`: `jid null` without active companion
  registration refuses to connect at all (`connect/ignored/jid null`).

The live value from the first-login capture: `<success pn='6615@s.whatsapp.net'
lid='4312@lid' ...>` — the phone JID of this installation is `6615@s.whatsapp.net`,
so `6615` went into username. This is a first pn-JID login: the `product` field is
not set in this case (it is set only on LID login).

username+static identity: the server matches the pair "the number from field 1 + the
client static key from `ClientFinish.static`". Changing either of the two already
means a different account.

## passive (field 3, bool)

Constant: `PASSIVE_FIELD_NUMBER = 3` (`X/C1HS.java:42`). Set by the case 6559 filler
(`X.1J8`, `X/C1US.java:20318-20323`) — unconditionally, it is always a bool:

```java
boolean z = c1hm.A09;          // XmppLoginContext.passive
c1hs2.bitField0_ |= 2;
c1hs2.passive_ = z;
```

The value is computed once in `C0SW.A0v` (`X/C0SW.java:1635`) and arrives in the
context:

```java
boolean z5 = (zA02 || ((C04930Mc) this.A0T.get()).A02()
              || (!z2 && !zA0P && !zA1J && !zA09)) ? false : true;
```

Passive mode = "I am connecting, but I will not immediately pull everything" (the
server sends data only on request). The exact logic:

`passive = false` if at least one of the following holds:

| Condition | Class | Meaning |
|---|---|---|
| `zA02` | `C03360Fw.A02()` (`X/C03360Fw.java:20`) | an active companion registration phase is in progress: pref `companion_registration_state` in values 2..6 or 10..17 |
| `Mc.A02()` | `C04930Mc.A02()` (`X/C04930Mc.java:34`) | PAA-link is active: pref `paa_link_mode_enabled=true`, mode is not `DEPENDENT` and status is not `COMPLETED` |
| `!z2 && !zA0P && !zA1J && !zA09` | argument of `A0v` + `C1E5.A0P()` + `C018208l.A1J()` + `C1F2.A09()` | there is not a single reason to be passive (see below) |

`passive = true` only when: no companion registration, no PAA-link, and at the same
time there is at least one reason:

- `z2` — `forcePassiveMode`, an argument of the call `A0v(C26191Dt, String, boolean z, boolean z2)`
  (an explicit passive connect; in the log `ConnectionThread/connect/start ... forcePassiveMode=`,
  `X/C0SW.java:1238-1243`);
- `zA0P` — there are preacks: the queue is not empty (`C1E5.A0P()`, `X/C1E5.java:578` —
  "more than zero outgoing messages in the queue");
- `zA1J` — pref `signal_protocol_store_is_new` = true (`C018208l.A1J()`,
  `X/C018208l.java:6790`) — a fresh signal store, nothing to pull;
- `zA09` — "passive based on queue size" (`C1F2`, `enablePassiveModeBasedOnQueueSize`
  in the log).

So on a phone's first login (no queue, the store has already been created at
`/v2/register`, no force) — `passive=false`. The same is visible in the capture:
right after `<success>` the client sends `<presence type='available'/>` (02:48:52.462),
not a passive presence.

The log line where the value is visible live (before encryption) — `X/C0SW.java:1643-1655`:

```text
ConnectionThread/connect: SEND <handshake_payload connect_attempt_count=N login_count=L
  passive=true/false session_id=S short_connect=B connect_type=T connect_reason=R />
  hasPreacks=... enablePassiveModeBasedOnQueueSize=...
```

## Link with the rest of the payload

- `passive=true` affects fields 45/46 (`preacksCount`, `processingQueueSize`): the
  queue snapshot `A1S(z5)` is taken always under AB 20485, or under AB 22413 && passive
  (`X/C0SW.java:738-744`) — see `misc-fields.md`.
- `username` is absent → fields 18/19 work in its place (companion).
- A passive session does not send `<presence type='available'/>` after `<success>` —
  by this sign passivity is visible in any FunXmpp capture.

## Where to look

- `X/C1US.java:20686-20734` (X.1JV, username), `20295-20382` (X.1J8, passive).
- `X/C0SW.java:1631-1660` — passive computation, context assembly, handshake_payload log.
- `X/C03360Fw.java:20`, `X/C04930Mc.java:34`, `X/C1E5.java:578`, `X/C018208l.java:6790`.
- Enum `product`: `X/C1JY.java`.
