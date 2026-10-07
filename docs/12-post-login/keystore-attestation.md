# Keystore attestation: `<ib><keystore_attestation>`

WhatsApp 2.26.35.75 (263507522). Wire: live capture of the first login; the node is sent at 02:48:52.769 — 24 ms after the server's `<safetynet><integrity nonce>` (.745). The body is a single base64 blob (4556 characters = **3417 bytes of DER**): four X.509 certificates back to back, with no separators.

## The chain of this session (Samsung TEE)

The order on the wire is root → TEE → TEE → leaf. This order is produced by `X/C13D.A03`: Android returns the chain as `[leaf … root]`, and the loop writes it from the end:

```java
for (int length = certificateChain.length - 1; length >= 0; length--) {
    byteArrayOutputStream.write(certificateChain[length].getEncoded());
}
```

| # | Bytes | Subject | Signer | Validity window |
|---|---|---|---|---|
| 1 | 1312 | serialNumber `f92009e853b6b045`, RSA | self-signed | 2019-11-22 20:37:58 … 2034-11-18 20:37:58 |
| 2 | 919 | OU=TEE, serialNumber `70912df46104faae947864e5804f1f80d`, EC | #1 (the root) | 2020-09-28 20:25:02 … 2030-09-26 20:25:02 |
| 3 | 503 | OU=TEE, serialNumber `91bb64f6f0dfcbe26de63a87ddc6c4e1`, EC P-256 | #2 | 2020-09-28 20:26:58 … 2030-09-26 20:26:58 |
| 4 (leaf) | 683 | `CN=Android Keystore Key`, EC P-256 | #3 | 2021-06-16 19:18:00 … 2031-06-14 19:18:00 |

Live check of the dates: 1312+919+503+683 = 3417. The TEE certificates were issued in 2020, the leaf has a ten-year window 2021–2031; this is characteristic Samsung keymaster behavior: the leaf's date is not tied to the moment of the request; binding to the session is done through the challenge (below).

## The Keymaster extension 1.3.6.1.4.1.11129.2.1.17

Present only on the leaf, 310 bytes. KeyDescription parsing of this session:

| Field | Value |
|---|---|
| attestationVersion | 3 |
| attestationSecurityLevel | 1 (TEE) |
| keymasterVersion | 4 |
| keymasterSecurityLevel | 1 (TEE) |
| attestationChallenge | 41 bytes |
| uniqueId | empty (OCTET STRING of length 0) |
| AuthorizationList | purpose 3, alg EC, digest SHA-256/SHA-512, all applications (`allApplications` = SHA-256 of the `com.whatsapp` signature) |

### Challenge — the binding to the moment of login

The 41-byte challenge, byte by byte from the live leaf:

```text
00 00 00 00 6a 98 b5 e4   8 байт big-endian = 1788392932
1f                        один байт 0x1f
0f 6f b9 70 6a 1f a8 60 41 8a b2 b3 fc df d6 10
bf 46 ea 97 7f 72 a1 6b 0d 5b 2e 72 c9 0e dc 36   32 байта случайные
```

`1788392932` is exactly the `t` attribute of this session's `<success>` (2026-09-02 23:48:52 UTC). The random 32 bytes equal neither the safetynet nonce (66B) nor the gpia nonce (57B). The framework assembles it in `X/C13D.A02` when creating the pair:

```java
long jA00 = C08A.A00(c08a) / 1000;          // часы, сдвинутые на серверное t из <success>
ByteBuffer byteBufferAllocate = ByteBuffer.allocate(bArr2.length + 8 + 1);
byteBufferAllocate.order(ByteOrder.BIG_ENDIAN);
byteBufferAllocate.putLong(jA00);
byteBufferAllocate.put((byte) 31);          // 0x1f
byteBufferAllocate.put(bArr2);              // SecureRandom, длина из AB 2078
certificateNotAfter.setAttestationChallenge(byteBufferAllocate.array());
```

If the caller did not pass bytes, `bArr2` is `SecureRandom` of the length from AB flag **2078** (32 bytes here). The default pair algorithm is EC — flag **2076**; digests SHA-256 and SHA-512, `setUserAuthenticationRequired(false)`. On the new API, `setDevicePropertiesAttestationIncluded(true)` is set; on a `ProviderException` the flag is cleared and generation is retried.

## Enablement and aliases

- The whole chain is sent only when AB flag **1934** is enabled. Otherwise `C13D.A03` returns `null` and there is no node on the wire.
- Key pair aliases: `my_personal_mini_pony_static` and `my_personal_mini_pony`; a suffix from the prefs `ka_key_store_*_alias_suffix` may be appended to them.
- The `keystore_attestation` tag exists only in the FunXMPP dictionary (`X/AbstractC28731Rb`); there is no literal in the native `libwhatsapp.so` dump — the encoder knows the token, while the content is prepared by the Java KeyStore.

## Why the chain arrives within a 24 ms window

Between the safetynet nonce (.745) and `<ib><keystore_attestation>` (.769) there is no other Java call that touches the KeyStore: the chain is read within the same window of the periodic integrity request (`DTA` case 10). `DTA` case 10 does not read the chain itself — the aliased pair already exists (created earlier; the challenge of each generation binds it to a specific second of the login); it is read by the adjacent path `C13D.A03`, and the result is sent in a shared `<ib>` alongside the integrity_payload (.780).

Visibility boundary: the TEE native code (the keymaster daemon itself) is not in the `libwhatsapp.so` dump. What is visible from the client: the challenge was assembled in the second of `<success>` in the form of `C13D.A02`, and this Samsung's leaf certificate window is 2021–2031, not "now plus hours" (AB 2079 sets the lifetime of future generations; here the previously issued window applies).

## A different profile: Google remote provisioning (live fact from 2.26.37.73)

On a fresh capture of 2.26.37.73 (263707322, the same SM-A325F, 2026-09-28), the chain of the same node is already Google remote provisioning:

```text
Key Attestation CA → Droid CA2 → Droid CA3 (TEE, serial ccad9877…) → leaf CN=Android Keystore Key
```

The node format, the "from the end" order (`C13D.A03`), and the 41-byte challenge are the same; only the chain issuer differs. Conclusion for analysis: the chain's issuer profile depends on the device's provisioning mode (Samsung TEE versus RKP); this changes nothing in the client code.

## In brief: what the server verifies

1. The chain from the leaf to a trusted root (Samsung root or Google RKP).
2. `attestationSecurityLevel=TEE` (1) — the key lives in the TEE, not in software.
3. Challenge: the first 8 bytes = the `t` of the last `<success>` — the attestation is fresh and belongs to this login session, not reused.
4. `allApplications`/`applicationId` = `com.whatsapp` — the pair was created by this application.
5. `uniqueId` is empty — the client did not request a unique ID.

## Where to look in the code

- Challenge and pair generation: `X/C13D.java`, method `A02`.
- Chain output from the end: `X/C13D.java`, method `A03` (returns `null` without AB 1934).
- AB flags: **1934** (node enablement), **2076** (EC algorithm), **2078** (length of the random part of the challenge), **2079** (generation window).
- Aliases and suffix: prefs `ka_key_store_*_alias_suffix`.
- Live wire: 02:48:52.769 (a blob of 4556 b64 chars).
- Related: `integrity-payload.md` — the second node of the same window.
