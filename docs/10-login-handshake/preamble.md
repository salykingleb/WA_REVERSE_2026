# Preamble: bytes on the socket before ClientHello

Build 2.26.35.75 (263507522). This document describes everything the client writes to
the chat socket OutputStream before the first HandshakeMessage protobuf frame, and how
it reads the response frames. Code: `X/C1KE.java`, `X/C1KV.java`, `X/C1KU.java`,
`X/AbstractC27051Jq.java`.

## 1. Edge header `ED\x00\x01` (optional)

The `C1KE` constructor (span `send_preamble`, the name from `X/C1KS.java:61` — code 27,
human-readable `SendPreamble`), before the Noise preamble, checks the
SharedPreferences key `routing_info`. If it is non-empty:

1. 4 bytes of the constant `C1KE.A0A = {69, 68, 0, 1}` (`X/C1KE.java:12`) are
   written — this is ASCII `E`, `D`, then `\x00\x01`: **`ED\x00\x01`** (hex `45 44 00 01`).
2. 3 bytes of the blob length are written, big-endian (`AbstractC27051Jq.A04(int)` →
   `{(byte)(i>>16), (byte)(i>>8), (byte)i}`, `X/AbstractC27051Jq.java:18-20`).
3. The blob itself is written: base64-decode of the `routing_info` value with flag 3
   (BASE64_DEFAULT | NO_WRAP | NO_PADDING are not set explicitly — flag 3 =
   `Base64.DEFAULT` in Android terms, without line wraps, like the other storage
   keys).

The complete wire format:

```
45 44 00 01 | LL LL LL | <routing_info blob, LL bytes> | 57 41 06 03 | ...
 E  D  \x00 \x01  BE length   edge routing blob          W  A  \x06 \x03
```

Where the blob comes from: the server sends it **after** a successful login, in the
node `<ib><edge_routing><routing_info>...</routing_info></edge_routing></ib>`
(live capture of the first login, 02:48:52.706:
`FunXmppDecoder.readNode() => <ib from='s.whatsapp.net'><edge_routing><routing_info></routing_info></edge_routing></ib>`).
The handler `C27871Nj` saves the blob to prefs; on the **next** connection it is sent
with this `ED` header, helping the load balancer pick the same edge. On the very first
login `routing_info` is empty, the `ED` header is not written at all — the first byte
of the connection is `W`.

## 2. Noise preamble `WA\x06\x03` (always)

Right after the edge header (or right away as the first one, if there is none) 4 bytes
`{87, 65, 6, 3}` = **`WA\x06\x03`** (hex `57 41 06 03`) are written. They are generated
by `C1KE.A04()` (`X/C1KE.java:153-159`):

```java
private byte[] A04() {
    if (this.A00 == 6) {
        return new byte[]{87, 65, 6, 3};
    }
    Log.e("NoiseSocket protocol version is not 5 or 6");
    return new byte[]{87, 65, 5, 3};
}
```

- The protocol version is stored in the field `C1KE.A00`; the constructor of this
  build always puts **6** there.
- Preamble byte layout: `W` `A` = the WhatsApp chat protocol, `0x06` = version 6,
  `0x03` = the framing subversion/revision.
- The version 5 branch (`WA\x05\x03`, hex `57 41 05 03`) is **unreachable** in this
  client: `A00` never takes a value other than 6. The branch code remained in `A04`
  in case of a different field value; the old `BW6`/`Bar` certificate scheme is also
  tied to the version 5 branch (the "ServerHello and certificate" chapter).
  Any value other than 5/6 is additionally punished with `Log.e("NoiseSocket protocol
  version is not 5 or 6")`.

## 3. HandshakeMessage framing — `C1KV` (write)

Each handshake message (ClientHello, ClientFinish) is serialized as the protobuf
`C1LV` and wrapped in `X/C1KV` (FilterOutputStream, `X/C1KV.java:8-34`):

| Element | Size | Meaning |
|---------|--------|----------|
| Body length | 3 bytes big-endian | `AbstractC27051Jq.A04(i2)` (`X/C1KV.java:13`) |
| Body | up to 2^24−1 bytes | the serialized HandshakeMessage protobuf (`X/C1KV.java:14`) |
| Limit | 16777216 | the check `if (i2 < 16777216)` (`X/C1KV.java:12`); on exceedance — the exception `C4N("data too large to write; length=" + i2)` (`X/C1KV.java:16-21`) |

After each frame — `flush()` (`X/C1KV.java:15`). The `write(int)` and
`write(byte[])` methods reduce to the same (`X/C1KV.java:25-33`).

## 4. Reading the response — `C1KE.A00` and the goaway marker `C1KU.A01`

The response frame (ServerHello) is read by `C1KE.A00()` (`X/C1KE.java:80-112`) in
four steps:

1. Span `read_server_hello` (`C1KS` code 22, `X/C1KE.java:84`: `C1KJ.A00(C02S.A0F, c1kj)`).
2. Exactly 3 length bytes are read: `byte[] bArr = new byte[3]; C1KU.A00(c1ku, bArr)`
   (`X/C1KE.java:86-87`). `C1KU.A00` is a buffer top-up loop; on EOF before
   completion: `IOException("Closed before read completed!")` (`X/C1KU.java:12-23`).
3. Comparison with the goaway marker `C1KU.A01 = {71, 79, 65}` (`X/C1KU.java:9`) —
   ASCII **`GOA`**. If it matches exactly — `throw new IOException() { // from class: X.1Rx }`
   (`X/C1KE.java:88-91`). This anonymous class `X.1Rx` is caught in
   `C0SW.A0v` as `C28931Rx`: the log `ConnectionThread/connect/socket/goaway` and
   `throw new C28861Rq(6, -1)` (`X/C0SW.java:1586-1592`) — the server politely asks
   to reconnect to another edge; ClientHello will not happen on this connection.
   Important: the marker is compared against the bytes read as the "length", that is,
   the server may send just `GOA` instead of a frame.
4. The body is read with the length `AbstractC27051Jq.A00(bArr)` — the 3-byte BE
   decoder: `(bArr[2]&255) | ((bArr[0]&255)<<16) | ((bArr[1]&255)<<8)`
   (`X/AbstractC27051Jq.java:8-10`, the call `X/C1KE.java:92-93`) — and parsed as
   `GeneratedMessageLite.parseFrom(C1LV.DEFAULT_INSTANCE, bArr2)` (`X/C1KE.java:94`).

A mandatory requirement for the frame: in the parsed `C1LV` the field 2 bit
(serverHello) must be set:

```java
if ((c1lv.bitField0_ & 2) == 0) {
    throw new IOException("Handshake message does not contain server hello!");
}
```

(`X/C1KE.java:95-97`). Bit 2 corresponds to the field `serverHello_` with the field
number 3 (`C1LV.SERVER_HELLO_FIELD_NUMBER = 3`, `X/C1LV.java:18`; the bitmask in
protobuf-lite follows the declaration order: clientHello=bit 1, serverHello=bit 2,
clientFinish=bit 3). If serverHello is present inside but empty —
`C1LX.DEFAULT_INSTANCE` is substituted (`X/C1KE.java:98-101`), and the parsing
fails further along the field processing.

After `A00` the socket remains in the "raw" XML node mode — the switch of the
framing is described in the success/failure chapter.

## 5. Summary of the client's first packet (first login)

For the first login (the server static is not saved, `routing_info` is empty) the
start of the client's stream looks like this:

```
57 41 06 03                       — preamble WA6.3
LL LL LL                          — length of protobuf HandshakeMessage{clientHello}
<handshake-message protobuf>      — ClientHello (ClientHello chapter)
```

For a re-connection with a saved edge:

```
45 44 00 01                       — edge header
LL LL LL <routing_info blob>      — saved edge routing
57 41 06 03                       — preamble
LL LL LL <HandshakeMessage>       — ClientHello of the resume variant (ClientHello chapter)
```

Hash mixing: the `WA\x06\x03` preamble is not only written to the socket, but is also
mixed into the Noise protocol hash — the initialization of the hash with the pattern
name and the mixing in of the preamble are covered in the "Noise modes" chapter (span
`init_cipher_full` = code 19, `init_cipher_resume` = code 20, `X/C1KS.java:49-50`).
