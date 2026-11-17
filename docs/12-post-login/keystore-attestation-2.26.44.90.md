# Keystore attestation in 2.26.44.90: builder vs sender

Detailed comparison of `keystore_attestation` between 2.26.35.75 and 2.26.44.90.
Builder: `X/17x`. Sender: `X/2Eu`. Wire data: live login traffic of 2.26.44.90.

## 1. Who builds and who sends

| | 2.26.35.75 | 2.26.44.90 |
|---|---|---|
| Builder | `X/C13D` | `X/17x` |
| Sending | inside the login flow, DTA case 10 | separate path `X/2Eu`: `att-stanza-custom` → `<ib><keystore_attestation>`, id **6** |
| Gate | AB 1934 | AB 1934 **and** AB 1935 |
| Throttling | none | AB 2067 + pref `ka_stanza_sent_ts` |
| Key alias prefix | AB 2074 | AB 2074 (`0x81a`) |
| Challenge random length | AB 2078 | AB 2078 (`0x81e`) |
| Key validity window | AB 2079 | AB 2079 (`0x81f`) |

> AB 2078 and 2079 are **not removed**. In 2.26.44.90 they are read through `A0Z`
> instead of `A0Y`, so a grep for `A0Y(2078)` returns nothing. Actual occurrences:
> `X/17x:1336` (`0x81e`) and `X/17x:605`, `:1133` (`0x81f`).

## 2. Challenge

`17x.A00(Integer, byte[])` (`X/17x:1315-1421`):

```
# if p2 == null: generate the random part
invoke-static {v0}, LX/182;->A00(LX/182;)Ljava/security/SecureRandom;
const/16 v0, 0x81e            # AB 2078 = random part length
new-array v9, v0, [B
invoke-virtual {v1, v9}, Ljava/util/Random;->nextBytes([B)V

# assemble the challenge
invoke-static/range {v25}, LX/08q;->A00(LX/08q;)J   # corrected clock, ms
div-long v0, v0, v16                                 # v16 = 0x3E8 → seconds
invoke-static {v8}, Ljava/nio/ByteBuffer;->allocate(I)  # len(random) + 9
invoke-virtual {v11, v8}, Ljava/nio/ByteBuffer;->order(Ljava/nio/ByteOrder;)  # BIG_ENDIAN
invoke-virtual {v11, v0, v1}, Ljava/nio/ByteBuffer;->putLong(J)               # 8 bytes of time
const/16 v0, 0x1f
invoke-virtual {v11, v0}, Ljava/nio/ByteBuffer;->put(B)                        # 0x1F
invoke-virtual {v11, v9}, Ljava/nio/ByteBuffer;->put([B)                       # random / p2
invoke-virtual {v7, v0}, Landroid/security/keystore/KeyGenParameterSpec$Builder;->setAttestationChallenge([B)
```

Result: `challenge = uint64BE(seconds) ‖ 0x1F ‖ random(AB 2078)` — the same layout as
in 2.26.35.75.

**New:** when `p2 != null` the random part is not generated; the supplied array is used.
The custom-stanza path (`X/2Eu:1036`) calls `17x.A04(17x, Integer, byte[])` with
bytes obtained from the `att-stanza-custom` exchange, so the challenge part may be
server-provided.

Live capture (2.26.44.90): `p2` was **32 bytes**, e.g.
`d9dec863ce3b7228b1c70205af5e0f7228e76c363dfeddf407a54e1987a1ba79`,
chain length **3420 B**.

## 3. Key validity window

`A0Z(2079)` is read twice — when computing `notBefore`/`notAfter` (`X/17x:605`) and in
the second pass (`X/17x:1133`). Semantics are unchanged: `now + 2079` seconds.

## 4. Alias

`0x81a` = AB 2074, dynamic alias prefix (`X/17x:1752`, `1886`). Base name `att-gen`
(`X/17x:203`), logs `blacknoise/generating key pair with alias=`.

## 5. Certificate chain

`17x.A03` walks the `[leaf … root]` array in reverse and writes it back to front — the
same as `C13D.A03`. The wire format is one blob without separators, root → … → leaf.

## 6. New sending path (`2Eu`)

```
17x.A07()                    # AB 1934
  └─ AB 1935 (0x78f)
       └─ pref ka_stanza_sent_ts + A0Z(2067) <= now/1000
            └─ 17x.A04(...)  # chain assembly (challenge from att-stanza-custom)
                 ├─ null → Log.w("blacknoise: got null attestation chain in custom stanza")
                 └─ new C0T8("keystore_attestation", chainBytes, null)
                      └─ new C0T8("ib", ...) → send id 6
                      └─ pref ka_stanza_sent_ts = now/1000
```

Lines: `X/2Eu:895-1000` (gates and throttling), `:1036` (assembly), `:1062` (node),
`:1056-1080` (`ib` wrapper and send).

## 7. Live capture (2.26.44.90, fresh registration)

```
[IN ] <success>
[OUT] <ib><unified_session id="127334115">
[ATTEST] mode=0 p2=d9dec863… (32B) -> chain 3420B
[OUT] <ib><keystore_attestation>  (3420B DER, 3082051c30820304a003020102…)
[OUT] <iq xmlns=w:stats type=set id=02><add t="1794914537"> + 1156B blob
```

- `p2` — 32 bytes supplied by the caller; `SecureRandom(AB 2078)` is not used on this path.
- Final challenge: `uint64BE(seconds) ‖ 0x1F ‖ p2(32B)` = 41 bytes.
- Chain: 3420 bytes (2.26.35.75 live capture gave 3417).
- The node is sent right after `<ib><unified_session>` and before `<iq xmlns=w:stats>`.
- No attributes: `<ib><keystore_attestation>` + one binary child with the DER.

Ab-flag summary for 2.26.44.90:

| AB | Hex | Where | Meaning |
|---|---|---|---|
| 2074 | `0x81a` | `X/17x:1752,1886` | dynamic alias prefix |
| 2078 | `0x81e` | `X/17x:1336` | challenge random part length |
| 2079 | `0x81f` | `X/17x:605,1133` | key validity window |
| 1934 | `0x78e` | `X/17x:3203` (`A07`) | attestation gate |
| 1935 | `0x78f` | `X/2Eu:912` | custom stanza gate |
| 2067 | `0x813` | `X/2Eu:972` | resend throttling |
