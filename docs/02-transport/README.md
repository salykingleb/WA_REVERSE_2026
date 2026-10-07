# Transport layer of /v2/code and /v2/register requests

**Stage A** · the `02-transport/` folder of the WhatsApp 2.26.35.75 (263507522) reverse-engineering documentation.

Request envelope: HTTPS/TLS, parameter encryption (ENC), signature (H), keystore attestation (Authorization), request_token and the exact order of parameters on the wire.

## Documents in this folder

| Document | Title |
|---|---|
| [Authorization.md](Authorization.md) | Authorization — header with the keystore attestation certificate chain |
| [ENC.md](ENC.md) | ENC — encryption envelope of all registration parameters |
| [H.md](H.md) | H — request signature (ECDSA-P256-SHA256 over the ENC string) |
| [http-transport.md](http-transport.md) | HTTPS_transport — common framework of the /v2/code and /v2/register registration HTTP requests |
| [params-order.md](params-order.md) | params_order — exact sequence of query fields for /v2/code and /v2/register |
| [request_token.md](request_token.md) | request_token — idempotency header of the registration request |

Navigation across the whole guide: [docs/README.md](../README.md). Start: [Flow overview](../01-flow-overview.md).
