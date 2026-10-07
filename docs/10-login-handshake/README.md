# Account login: Noise handshake

**Stage C** · the `10-login-handshake/` folder of the WhatsApp 2.26.35.75 (263507522) reverse engineering documentation.

Chat socket establishment: the WA\x06\03 preamble, Noise pattern selection (XX/IK/PQ), ClientHello, server certificate verification, ClientFinish, success/failure parsing.

## Documents in this folder

| Document | Title |
|---|---|
| [client-finish.md](client-finish.md) | ClientFinish: the client static key and the login payload |
| [client-hello.md](client-hello.md) | ClientHello: protobuf, state machine, full and resume variants |
| [connect-preconditions.md](connect-preconditions.md) | Connection preconditions: when the chat socket is opened at all |
| [noise-modes.md](noise-modes.md) | Noise modes: the full/resume fork, pattern names, PQ variants |
| [preamble.md](preamble.md) | Preamble: bytes on the socket before ClientHello |
| [server-hello-certificate.md](server-hello-certificate.md) | ServerHello: DH operations, PQ TLV and version 6 certificate verification |
| [success-failure.md](success-failure.md) | Login response: `<success>` and `<failure>`, client disconnects before the response |

Navigation for the whole guide: [docs/README.md](../README.md). Start: [Flow overview](../01-flow-overview.md).
