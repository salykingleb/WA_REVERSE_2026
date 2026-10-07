# ServerHello: DH operations, PQ TLV and version 6 certificate verification

Build 2.26.35.75 (263507522). Parsing of the server response after ClientHello.
Protobuf `X/C1LX.java`; handler `C1KE.A03` (`X/C1KE.java:145-151`, body not
decompiled — logic from the jadx fallback dump); certificate protobufs:
`X/BW5.java`, `X/BW4.java`, `X/C25957Baq.java`; issuer `X/HWN.java`.

## 1. ServerHello protobuf (`X/C1LX.java:15-20`)

| Field | Number | Type | Contents |
|------|-------|-----|------------|
| `ephemeral` | 1 | bytes | the server's X25519 ephemeral, plaintext |
| `static` | 2 | bytes | the server's encrypted static X25519 |
| `payload` | 3 | bytes | encrypted certificate (in PQ — TLV + certificate) |
| `extendedStatic` | 4 | bytes | PQ extension of the static |
| `paddingBytes` | 5 | bytes | padding |
| `extendedCiphertext` | 6 | bytes | the server's KEM part |

Mandatory: a frame without `serverHello` is rejected already at read time
(`Handshake message does not contain server hello!`, `X/C1KE.java:95-97`,
the "Preamble" chapter).

## 2. Step-by-step parsing (`C1KE.A03`)

The order of operations and their span names (`X/C1KS.java`):

| Step | ServerHello field | Operation | Span (code) |
|-----|------------------|----------|------------|
| 1 | `ephemeral` (1) | DH ee: the shared secret of client ephemeral × server ephemeral; the ephemeral is mixed into the hash (`C1KX.A02`/`A03`) | `ecdh_ee` (5) + `hash_sv_e` (16) |
| 2 | `static` (2) | Decryption `decrypt_sv_s` (2): yields the server's plaintext static X25519; DH es (client ephemeral × server static), mixing in | `decrypt_sv_s` (2) + `ecdh_es` (6) |
| 3 | `payload` (3) | Decryption `decrypt_sv_c` (3): classic — the certificate; PQ — first the TLV with the ML-KEM key, then the certificate | `decrypt_sv_c` (3) |
| 4 | — | Certificate verification | `validate_server_cert` (28) |

Steps 1–2 contribute two DH inputs into the key (ee, es) — the standard Noise XX/IK.
After the certificate is accepted, the client will add se in ClientFinish (the
ClientFinish chapter).

## 3. PQ TLV inside `payload`

In PQ mode a TLV chain lies in the decrypted `payload` before the certificate:

| Type | Length | Body | Error on violation |
|-----|-------|------|---------------------|
| `0x01` | = len("MLKEM512") = 8 | exactly the bytes `MLKEM512` (constant `C1KE.A0B`, `X/C1KE.java:13`) | `Unexpected KEM algorithm` / `Expected TLV header type 0x01` |
| `0x02` | exactly 800 | ML-KEM-512 public key (800 bytes) | `Expected TLV PK length 800` / `Expected TLV PK type 0x02` |

TLV format: 1 byte type + 2 bytes length (big-endian) + body. Any violation of
type/length/body — IOException → disconnect `C28951Rz` → `ConnectionThread/connect/
socket/disconnect/noise` (the error type from `C51075Mse.reason`), see the
success/failure chapter.

After parsing the TLV the client performs an **encapsulate** on the ML-KEM-512 key:
`C1KX.A07(KEMPublicKey)` (`X/C1KX.java:202-222`, span `encapsulate` (9)):
the ciphertext (field `Encapsulated.ciphertext`) is mixed into the hash and will
later go into `ClientFinish.extendedCiphertext`, while `sharedSecret` is immediately
mixed into the chain key (`C1KX.A00`, MixKey, `X/C1KX.java:75-80`).

## 4. The version 6 certificate — protobuf `BW5`

The current version's format (this client). `BW5` (`X/BW5.java:13-18`):

| Field | Number | Type |
|------|-------|-----|
| `leaf` | 1 (`LEAF_FIELD_NUMBER`) | message `BW4` |
| `intermediate` | 2 (`INTERMEDIATE_FIELD_NUMBER`) | message `BW4` |

`BW4` (`X/BW4.java:14-19`): `details` (1, bytes) + `signature` (2, bytes).
The `details` of both nodes are parsed as `Baq` (`X/C25957Baq.java:15-25`):

| Baq field | Number | Type | Meaning |
|----------|-------|-----|-------|
| `serial` | 1 | int64 | certificate serial number |
| `issuerSerial` | 2 | int64 | issuer serial number (0 = root) |
| `key` | 3 | bytes | subject public key (X25519) |
| `notBefore` | 4 | int64 | validity from |
| `notAfter` | 5 | int64 | validity until |

### Verification rules (all mandatory)

| # | Rule | Executor code |
|---|---------|-----------------|
| 1 | The intermediate has `serial` and `issuerSerial` set; the leaf — `key` and `issuerSerial` | parsing details → Baq |
| 2 | `intermediate.serial == leaf.issuerSerial` (the leaf is signed by exactly this intermediate) | `C1KE.A03`, span `validate_server_cert` |
| 3 | `intermediate.issuerSerial == 0` (the intermediate was issued by the root, not by yet another intermediate) | same place |
| 4 | `leaf.key` is **byte-for-byte equal** to the server static X25519 decrypted in step 2; otherwise — `wa6: noise certificate key does not match proposed server static key` | same place |
| 5 | The leaf signature (`BW4.signature` of the leaf) is verified with the key `intermediate.details.key` | same place |
| 6 | The intermediate's signature is verified with the embedded issuer key `WhatsAppLongTerm1` from `X/HWN.java` | the `HWN.A00` map |
| 7 | Any bad signature → `wa6: invalid signature on noise certificate`; a missing intermediate key → `wa6: intermediate cert key is missing` | same place |

`X/HWN.java:11-17`: the issuer map contains a single entry
`"WhatsAppLongTerm1" → C27011Jk(bArr)`, where `bArr` (field `HWN.A01`) —
32 bytes (decimal `{20, 35, 117, 87, 77, 10, 88, 113, 102, -86, -25, 30,
-66, 81, 100, 55, -60, -94, -117, 115, -29, 105, 92, 108, -31, -9, -7, 84,
93, -88, -18, 107}`, hex `14 23 75 57 4D 0A 58 71 66 AA E7 1E BE 51 64 37 C4
A2 8B 73 E7 69 5C 6C E1 F7 F9 54 5D A8 EE 6B`) — the long-term WhatsApp root
public key embedded in the APK.

Failure of any check — the exception `C1S0` (class X.1S0; the file was not exported
by jadx due to a name register collision, referenced via `X/C0SW.java:1606`); outside,
in `C0SW.A0v` this is the branch:

```java
} catch (C1S0 e8) {
    Log.w("ConnectionThread/connect/socket/invalid-certificate-exception", e);
    c1fb.A0C();
    throw new C28861Rq(10, -1);
}
```

(`X/C0SW.java:1606-1610`) — a type 10 login error. The wrapper `C51075Mse`
(`X/C51075Mse.java:16-27`) for `C1S0` yields `reason = 6` (a telemetry report).

## 5. The version 5 branch — `BW6`/`Bar` (unreachable in this build)

The `C1KE` constructor always sets version 6, but the single-certificate parsing
code remains: protobuf `BW6` (`X/BW6.java:14-19`) — `details` (1) +
`signature` (2); details are parsed as `Bar` (`X/C25958Bar.java:16-27`):

| Bar field | Number | Type |
|----------|-------|-----|
| `serial` | 1 | int64 |
| `issuer` | 2 | string |
| `expires` | 3 | int64 |
| `subject` | 4 | string |
| `key` | 5 | bytes |

Version 5 rules: `issuer` must be a key of the `HWN.A00` map (that is,
`WhatsAppLongTerm1`); the `details` signature must verify with the key from `HWN`;
`key` must match the server static byte-for-byte; if `expires` is set —
it must not be earlier than the client's current time, otherwise
`noise certificate expired`.

## 6. Side effect of step 2 — persist

The accepted server static (and, with PQ, the key from the TLV after encapsulate) is
remembered for future resume connections: `C1KE.A08()` returns `C1KC` from the final
context (`X/C1KE.java:177-182`), `C0SW.A1c` passes it to
`C0SW.A1d(persistServerStaticKeys)` (`X/C0SW.java:1129-1131`, `791-807`),
the write to prefs — `AnonymousClass139.A0G/A0H` (keys `server_static_public` /
`server_static_pq_public`, span `persist_sv_s` (21)). Until the login fully
succeeds the key lives in memory; the next connection with it will go resume-IK
(the "Noise modes" chapter, decision table).

## 7. Summary of transitions on ServerHello errors

| Event | Exception | Handling in `A0v` | Login error type |
|---------|------------|-------------------|-------------------|
| TLV violations (`Unexpected KEM algorithm`, `Expected TLV ...`), other noise | `C28951Rz` | `ConnectionThread/connect/socket/disconnect/noise` + `c1fb.A0C()` (`X/C0SW.java:1437-1442`) | from `C51075Mse.reason` (4/5) |
| Certificate failed verification | `C1S0` | `invalid-certificate-exception` + `c1fb.A0C()` (`X/C0SW.java:1606-1610`) | 10 |
| The `GOA` marker instead of the frame length | `C28931Rx` (`X.1Rx` from `X/C1KE.java:88-91`) | `ConnectionThread/connect/socket/goaway` (`X/C0SW.java:1586-1592`) | 6 |
