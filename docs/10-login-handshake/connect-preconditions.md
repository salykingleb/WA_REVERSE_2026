# Connection preconditions: when the chat socket is opened at all

WhatsApp Android build 2.26.35.75 (versionCode 263507522). This is about the chat TCP
socket (port 443/5222, see chapter 02) on which the Noise handshake runs. Before the
first ClientHello byte the client passes a chain of refusals in `C0SW.A0v`. All
references point to the jadx decompilation of the app.

## 1. Entry point — `C0SW.A0v`

`X/C0SW.java:1198` — the method `A0v(C26191Dt, String, boolean z, boolean z2)` of the
class `C0SW` (this is `ConnectionThread`, a `HandlerThread` subclass). Arguments:
`z` = available (the outgoing queue is not empty), `z2` = forcePassiveMode. The method
runs in an attempt loop (`while(true)`, `X/C0SW.java:1307`); each iteration is a
new `attempt` (`c1fb.A05()`), whose state is written to the log
`ConnectionThread/connect: connecting; attempt=... state=...` (`X/C0SW.java:1403-1408`).

First the DI providers are read: the 6555 set (fill set) and 6563 (`X/C0SW.java:1219-1221`),
the companion flag `C03360Fw.A02()` (`zA02`, `X/C0SW.java:1222`) and the JID from `C08X.Aom()`
(`X/C0SW.java:1223-1224`).

## 2. Full table of pre-handshake refusals

The checks run strictly in this order; the first condition that fires terminates the
method (`return`), the socket is not opened, ClientHello is not built.

| # | Condition | jadx line | Log | Outcome |
|---|---------|-------------|-----|------|
| 1 | JID null (`C08X.Aom()`), this is not companion registration (`zA02 == false`) and the phone-JID `C08X.Aoj()` is also null | `X/C0SW.java:1226-1235` | `ConnectionThread/connect/ignored/jid null and not in companion reg` (Log.e) | exit without a callback |
| 2 | A live session already exists: `C06350Rv.A01()` (field `A1C`) | `X/C0SW.java:1245-1249` | `ConnectionThread/connect/already-connected` | exit |
| 3 | Device clock rejected: `C0AM.A02()` | `X/C0SW.java:1257-1262` | `ConnectionThread/connect/not-allowed/clock` | exit + callback `this.A1B.BdT()` |
| 4 | Previous login marked as failed (`C03380Fy.A0M()`) and this is not a companion (`!zA02`) | `X/C0SW.java:1263-1266` | `ConnectionThread/connect/not-allowed/login-failed` | exit |
| 5 | Build is stale: `C0AM.A01()` | `X/C0SW.java:1267-1271` | `ConnectionThread/connect/not-allowed/software-expired` | exit + callback `this.A1B.C37()` |
| 6 | Quit flag raised: `this.A1E.A01()` | `X/C0SW.java:1272-1275` | `ConnectionThread/connect/not-allowed/quit-flag-set` (via `A1R`, which stops the HandlerThread: `X/C0SW.java:726-736`) | exit, the thread terminates |

Notes on the table rows:

- **clock** (`C0AM.A02`): the system time is outside the allowed window (the server
  skew is too large). The `BdT` callback shows the "wrong clock" screen.
- **software-expired** (`C0AM.A01`): the build's lifetime has expired; the `C37` callback
  leads to the mandatory update screen.
- **jid null**: the only legal way forward with a null JID is the `zA02` mode
  (companion device registration); then `userJidAom` stays null and goes into
  `c1fb.A06(userJidAom)` (`X/C0SW.java:1625`), and the `username` field
  is not set in ClientPayload.

## 3. `ConnectionThread/connect/start` and network selection

If none of the refusals fired:

1. `Log.i("ConnectionThread/connect/start jid=<jid> available=<z> forcePassiveMode=<z2>")`
   — `X/C0SW.java:1237-1244`. This is the first positive marker: the login sequence
   has started.
2. Reset of the previous state: `this.A08 = null`, `C1E5.A0E()`, `C1E8.A00()`,
   `C1EB.A09()` (`X/C0SW.java:1250-1256`).
3. `Log.i("ConnectionThread/connect")` (`X/C0SW.java:1276`), `C249418s.A0L()`,
   the callback `C0S1.onConnecting()` (`X/C0SW.java:1279-1280`).
4. If the old socket `this.A07` is still alive (`isClosed() == false`) — forced
   closing via `A0T()` (`X/C0SW.java:1282-1285`).
5. **Network selection** (`X/C0SW.java:1286-1302`):
   - the host list `((C1Ew) this.A0M.get()).A01()`;
   - `Network network = (Network) this.A1L.getAndSet(null)` — if someone previously
     put a network-override, the log `ConnectionThread/connect/using_network_override`
     (`X/C0SW.java:1287-1290`);
   - the attempt context `C1FB` is assembled (constructor
     `X/C0SW.java:1302`): the network,
     socket providers, the auth-key-store `AnonymousClass139`, `C08X` (JID),
     `C018208l` (prefs), `C08A` (time), `str` (the connect reason from the caller),
     the host list, `Random`, the companion flag `zA02`.
6. `TrafficStats.setThreadStatsTag(1)` (`X/C0SW.java:1306`) and entry into the attempt loop.

Inside the loop (`X/C0SW.java:1307-1411`):

- `c1fb.A0F()` — whether there is still a right to try (attempt/host counter);
- with a blocked network (`C26441Ey.A08()`) the log gets
  `ConnectionThread/connect: Network blocked, skipping connection sequence attempt=... state=...`
  (`X/C0SW.java:1313-1320`);
- connection and streams: `C1HE c1heA09 = c1fb.A09(); this.A07 = c1heA09.A00();
  InputStream ... = c1heA09.A01(); OutputStream ... = c1heA09.A02();`
  (`X/C0SW.java:1411-1415`) — this is the raw TCP socket before any Noise;
- `c1hh = new C1HH(this)` — the future reader handler (`X/C0SW.java:1416`).

## 4. Handshake invocation `A0S` → constructor `C1KE`

Further along the flow: `C1FB.A06(userJid)` loads the keys (see below), `C1HN.A00(...)`
assembles the ClientPayload (`X/C0SW.java:1627-1640`), and `C0SW.A0S` performs the
Noise handshake.

`X/C0SW.java:235-256`:

```java
private C1KE A0S(C1FP c1fp, C1HS c1hs, InputStream inputStream,
                 OutputStream outputStream, Integer num, C1KD c1kd) throws IOException {
    ...
    C1KE c1ke = new C1KE(c1kf, c242015q, c08a, c1hs, inputStream, outputStream,
                         c1kd, new C1KG(num, ((C26441Ey) ...).A05()),
                         ((C26441Ey) ...).A06());
    Log.i("ConnectionThread/performHandshake: completed noise handshake; sessionId=" + c1fp.A07);
    ...
}
```

Parameters of the `C1KE` constructor (`X/C1KE.java:72`): `C1KF` (host selection), `C242015q`
(logger), `C08A` (time), `C1HS` (ClientPayload), the socket InputStream/OutputStream,
`C1KD` (key set: client static pair + server static keys), `C1KG` (PQ config:
`new C1KG(num, C26441Ey.A05())`, where `num` is the pqMode switch, and `A05()` is
pqProtocolVariant), boolean `C26441Ey.A06()`.

The `C1KE` constructor was not decompiled by jadx in normal mode (see the warning in
`X/C1KE.java:72-78`); its logic (preamble, full/resume fork, sending
ClientHello/ClientFinish) has been reconstructed and split across the chapters
of this section (preamble, Noise modes, ClientHello, ServerHello, ClientFinish).
The constructor itself sets protocol version 6: the field `A00 = 6`, and `A04()`
(`X/C1KE.java:153-159`) returns the preamble bytes `{87, 65, 6, 3}` = `WA\x06\x03`;
for any value other than 6 — `Log.e("NoiseSocket protocol version is not 5 or 6")`
and a return of `{87, 65, 5, 3}` = `WA\x05\x03` (an unreachable branch, see the
"Preamble" chapter for details).

After `A0S` returns, control is received by `A0x` (`X/C0SW.java:885-901`), which
starts reading nodes (the `C1OV` callback, span `C02S.A0G`), and `A0y` waits for
`success` / `failure` (the "success/failure" chapter).

## 5. Key loading `C1FB.A06/A0B` and the type 8 error

`C1FB.A0B()` (`X/C1FB.java:522-575`) reads the `AnonymousClass139` store
(prefs `keystore`):

| What | Key in prefs | Code |
|-----|--------------|-----|
| Server static X25519 | `server_static_public` (base64, flag 3) | read `X/C1FB.java:530-545`; write `AnonymousClass139.A0G` `X/AnonymousClass139.java:464-473` (`Log.i("saving server static public key")`) |
| Server static ML-KEM-512 | `server_static_pq_public` (base64, flag 3) | read `X/C1FB.java:547-561`; write `AnonymousClass139.A0H` `X/AnonymousClass139.java:475-488` |
| Client static pair (authkey) | `client_static_keypair_enc` (+ `_pwd_enc`) | `C1KB` from `AnonymousClass139.A0C()` (`X/AnonymousClass139.java:417-423`) |

The result is `C1KD(clientStaticKeys=C1KC|null, clientStaticPair=C1Jp)`
(`X/C1KD.java:36-39`). If the client pair could not be read:

```java
Log.e("ConnectionThread/connect/failed to load auth key, postponing login");
throw new IOException() { // from class: X.1Ry
```

(`X/C1FB.java:570-573`). The anonymous IOException class `X.1Ry` is caught in `A0v`
as `C28941Ry`: the log `ConnectionThread/connect/socket/disconnect/authKey` and
`throw new C28861Rq(8, -1)` (`X/C0SW.java:1593-1600`) — this is a client-side login
disconnect of type 8, still before ClientHello.

## 6. Class map of the section

| Class | Role |
|-------|------|
| `X/C0SW` | ConnectionThread: refusals, the attempt loop, handshake invocation, success/failure parsing |
| `X/C1KE` | NoiseSocket: preamble, frames, the full/resume fork (`X/C1KE.java`) |
| `X/C1KD` | Key wrapper: `C1KC serverStaticKeys` + `C1Jp clientLoginKeyPair` (`X/C1KD.java:15-17`) |
| `X/C1KC` | `ServerStaticKeys(serverStaticPublicKey, serverStaticPQPublicKey)` (`X/C1KC.java:37-52`) |
| `X/AnonymousClass139` | AuthKeyStore: prefs `keystore`, static key read/write |
| `X/C1KG` | `NoisePQConfig(pqMode, pqProtocolVariant)` (`X/C1KG.java`, toString) |
| `X/C26441Ey` | Source of the PQ settings from remote config (the "Noise modes" chapter) |
| `X/C28861Rq` | Login exception with the `type` and `serverErrorCode` fields (`X/C28861Rq.java:5-27`) |
