# Noise modes: the full/resume fork, pattern names, PQ variants

Build 2.26.35.75 (263507522). The "full handshake versus resume" fork and the choice
of the Noise pattern name. Code: `X/C1KE.java`, `X/C1KX.java`, `X/C1KG.java`,
`X/C1KH.java`, `X/C1KI.java`, `X/C26441Ey.java`, enum `X/C1LT.java`.

## 1. PQ configuration — where it comes from

The owner of the settings is `C1KG` (`X/C1KG.java`), `toString`:
`NoisePQConfig(pqMode=..., pqProtocolVariant=...)`. It is created in `C0SW.A0S`
(`X/C0SW.java:244`): `new C1KG(num, C26441Ey.A05())`.

- **pqMode** (`C1KG.A00`): values from `X/C1KH.java:6-15`:

  | Value | Name (C1KH.A00) |
  |----------|----------------|
  | 0 | `NOISE_PQ_MODE_NONE` |
  | 1 | `NOISE_PQ_MODE_ENABLE` |
  | 2 | `NOISE_PQ_MODE_ENFORCE` |

  The source is remote config: `C26441Ey.A04()` (`X/C26441Ey.java:32-46`) maps
  the value of the `Ez.A0i` flag: 1 → ENABLE(1), 2 or 3 → ENFORCE(2), otherwise → NONE(0).

- **pqProtocolVariant** (`C1KG.A01`): values from `X/C1KI.java:6-16` (the methods
  `A00`/`A01` are identical):

  | Value | Name (C1KI.A00) |
  |----------|----------------|
  | 0 | `IKKEM` |
  | 1 | `IKKEM_FS` |
  | 2 | `XXKEM_EPH_IKKEM2` |
  | 3 | `XXKEM_EPH_ONLY` |

  The source is the remote config `C26441Ey.A05()` (`X/C26441Ey.java:48-64`): a string
  with key 34536, default `"IKKEM"`; the strings `"IKKEM"`→0, `"IKKEM_FS"`→1,
  `"XXKEM_EPH_ONLY"`→3, `"XXKEM_EPH_IKKEM2"`→2.

Inside the `C1KE` constructor the PQ config lives in the field `A05` (`C1KG`); the
decryption of the server keys is `C1KC` (`A00` = X25519 `C27011Jk`, `A01` =
`KEMPublicKey` ML-KEM, `X/C1KC.java:49-52`).

## 2. The full / resume fork — `C1KE.A01`

`X/C1KE.java:114-128`:

```java
private Integer A01(C1KC c1kc) {
    if (c1kc != null) {
        C1KG c1kg = this.A05;
        if (c1kg.A00 != C02S.A00) {              // pqMode != NONE
            Integer num = c1kg.A01;
            if (num != C02S.A0N) {               // variant != XXKEM_EPH_ONLY
                if (num != C02S.A0C && c1kc.A01 == null) {  // != XXKEM_EPH_IKKEM2 and no PQ key
                    Log.w("NoiseSocket/handshake missing serverStaticPQ forcing full handshake");
                }
            }
        }
        return C02S.A01;                          // resume
    }
    return C02S.A00;                              // full
}
```

Return codes (`X/C02S.java:6-8`): `A00=0` full, `A01=1` resume. The full decision
table (the constructor's check order; jadx did not lift the constructor itself — the
logic is from the jadx fallback dump + the decompiled `A01`):

| # | Conditions | Mode | Log/note |
|---|---------|-------|----------------|
| 1 | `C1KC == null` (server static not saved) | **full XX** | first login: static appears only after a successful handshake (`AnonymousClass139.A0G`, `X/AnonymousClass139.java:464`) |
| 2 | PQ NONE + static present | **resume IK** | the classic re-login |
| 3 | PQ enabled, variant `XXKEM_EPH_ONLY` (3), static present | still **full XX** | the ephemeral-only KEM does not reuse the server static |
| 4 | PQ enabled, static present, but `serverStaticPQ == null`, variant ≠ `XXKEM_EPH_IKKEM2` | **full XX** | log `NoiseSocket/handshake missing serverStaticPQ forcing full handshake` (`X/C1KE.java:121`) |
| 5 | PQ enabled, static present (PQ key present, or variant `XXKEM_EPH_IKKEM2`) | **resume** | IKkem variants per the table below |

## 3. Full table of Noise pattern names

The name constants are the static block `X/C1KX.java:59-73`. The protocol field
`pqMode` in ClientHello is the enum `X/C1LT.java` (the value numbers are the protobuf
wire values).

| Client mode | Full handshake: pattern name (C1KX constant) | pqMode in ClientHello | Resume: pattern name | pqMode in ClientHello |
|---|---|---|---|---|
| PQ disabled (NONE) | `Noise_XX_25519_AESGCM_SHA256` (`A07`, `X/C1KX.java:17`) | not set | `Noise_IK_25519_AESGCM_SHA256` (`A0H`, `X/C1KX.java:61`) | not set |
| IKKEM (variant 0) | `Noise_XXkem_25519_MLKEM512_AESGCM_SHA256` (`A0E`) | `XXKEM` (1) | `Noise_IKkem+X25519+MLKEM512+AESGCM256+SHA256` (`A0F`) | `IKKEM` (5) |
| IKKEM_FS (variant 1) | `Noise_XXkem-FS_25519_MLKEM512_AESGCM_SHA256` (`A0C`) | `XXKEM_FS` (2) | `Noise_IKkem-FS+X25519+MLKEM512+AESGCM256+SHA256` (`A0D`) | `IKKEM_FS` (6) |
| XXKEM_EPH_IKKEM2 (variant 2) | `Noise_XXkemEph_25519_MLKEM512_AESGCM_SHA256` (`A08`) | `XXKEM_EPH` (9) | `Noise_IKkem2+X25519+MLKEM512+AESGCM256+SHA256` (`A0G`) | `IKKEM_2` (8) |
| XXKEM_EPH_ONLY (variant 3) | `Noise_XXkemEph_25519_MLKEM512_AESGCM_SHA256` (`A08`) | `XXKEM_EPH` (9) | resume is not performed — always full (row 3 of the decision table) | — |

The protobuf `pqMode` enum (ClientHello field 9, `X/C1LR.java:22`; enum `X/C1LT.java:7-17`):

| Value | Name | Value | Name |
|----------|-----|----------|-----|
| 0 | `HANDSHAKE_PQ_MODE_UNKNOWN` | 5 | `IKKEM` |
| 1 | `XXKEM` | 6 | `IKKEM_FS` |
| 2 | `XXKEM_FS` | 7 | `XXKEM_2` |
| 3 | `WA_CLASSICAL` | 8 | `IKKEM_2` |
| 4 | `WA_PQ` | 9 | `XXKEM_EPH` |

Fallback patterns (the server sent static in the resume response — the client
restarts into a full handshake, see the ClientHello chapter):
`Noise_XXfallback_25519_AESGCM_SHA256` (`A06`,
`X/C1KX.java:62`), `Noise_XXkemfallback_25519_MLKEM512_AESGCM_SHA256` (`A09`),
`Noise_XXkem-FSfallback_25519_MLKEM512_AESGCM_SHA256` (`A0B`),
`Noise_XXkemEphfallback_25519_MLKEM512_AESGCM_SHA256` (`A0A`)
(`X/C1KX.java:69-71`). The name of the state machine's fallback branch is
`client_resume` → `handle_server_fallback` (`X/C1KW.java:11, 23`).

## 4. Hash initialization with the pattern name — `C1KX.<init>`

Constructor `X/C1KX.java:224-264`:

1. Span code selection by pattern name (`X/C1KX.java:227-236`):
   - `A07/A0E/A0C/A0A8` (XX family) → span `C02S.A0B`;
   - `A0H/A0F/A0D/A0G` (IK family) → span `C02S.A0D`;
   - `A06/A09/A0B/A0A` (fallback family) → span `C02S.A0A`;
   - anything else → `IllegalArgumentException("Unknown handshake name")`.
2. If the name is longer than 32 bytes — first `SHA-256` of the name
   (`X/C1KX.java:243-246`):
   long names (`Noise_IKkem*...+SHA256`, 44+ bytes) are not visible on the handshake
   wire, but participate in the h-initialization through the hash. The 32-byte names
   (`Noise_XX_25519_AESGCM_SHA256` and `Noise_IK_25519_AESGCM_SHA256` — exactly 29
   bytes, padded to 32 with `Arrays.copyOf(..., 32)` zeros) go as is.
3. Internal hash initialization: `c1ky.A00 = bArr` (the pattern name is placed into
   the hash accumulator `C1KY`), then `c1ky.A00(bArr2)` — **mixing in the second
   blob**. That blob is precisely the protocol preamble: on a full handshake the span
   `init_cipher_full` (`C1KS` code 19, `X/C1KS.java:49`), on resume — the span
   `init_cipher_resume` (code 20, `X/C1KS.java:50`). In both cases the bytes
   `WA\x06\x03` get into the hash (the "Preamble" chapter), thanks to which the
   protocol transcript is bound both to the framing version and to the edge header.
4. Derivatives: `h` and the chain key `C27241Ko` (MixKey, `C1KX.A00`,
   `X/C1KX.java:75-80` — an HKDF expansion of 64 bytes into 32+32).

The standard mixing points (span names `X/C1KS.java:12-68`) referenced in
the other chapters: `hash_cl_e` (14) — the client ephemeral, `hash_sv_e` (16) and
`ecdh_ee` (5) — the server ephemeral, `decrypt_sv_s` (2) + `ecdh_es` (6) —
the server static, `encapsulate` (9) — KEM, `encrypt_cs` (10) and `ecdh_se` (7) —
the client static, `encrypt_login_payload` (11),
`persist_sv_s` (21),
`validate_server_cert` (28).

## 5. PQ objects

| Object | Class | Note |
|--------|-------|------------|
| ML-KEM-512 public key | `org.whispersystems.libsignal.kem.KEMPublicKey` | in `C1KC.A01`; on the wire in ServerHello.payload as TLV type 2, length 800 (ServerHello chapter) |
| Encapsulate | `C1KX.A07(KEMPublicKey)` (`X/C1KX.java:202-222`): `kEMPublicKey.A00()` → `Encapsulated{ciphertext, sharedSecret}`; the ciphertext is mixed into the hash, the secret — via MixKey `A00(this, sharedSecret)` | span: `encapsulate` (9) + `A05` |
| Algorithm name on the wire | `C1KE.A0B = "MLKEM512".getBytes(UTF_8)` (`X/C1KE.java:13`) | TLV type 1, length = len("MLKEM512") = 8 |
| The client's own ephemeral KEM key | generation span `gen_pq_eph_key` (12) | the `IKKEM_FS` and `XXKEM_EPH_IKKEM2` modes; sent as TLV type 3 (ClientHello chapter) |

## 6. What is visible in the live capture

The FunXmpp hook in the first-login capture sees the stream only after Noise
completes, so pattern names are not printed on the wire. Indirect evidence of the
first login: in `<success ... creation='1788392926'/>` creation is close to `t`
(a difference of 6 s) — a new account, the server static was absent before that,
which means the handshake went as full XX per row 1 of the decision table. For this
build, without remote config enabling PQ, that is `Noise_XX_25519_AESGCM_SHA256`
without the `pqMode` field.
