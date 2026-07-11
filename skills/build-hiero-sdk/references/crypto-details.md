# Cryptography Details

The exact constants and formats. These are protocol contracts — a single wrong
prefix or hash choice produces keys/signatures no other SDK or wallet accepts.

## Contents
1. Algorithms, byte lengths, DER prefixes
2. Signing: what is hashed, what is emitted
3. EVM address derivation
4. Mnemonics and HD derivation paths
5. String/bytes round-trip matrix and the 32-byte ambiguity trap
6. PEM/keystore
7. Porter gotchas checklist

## 1. Algorithms, byte lengths, DER prefixes

Two key types: **Ed25519** and **ECDSA secp256k1**.

DER prefix constants (hex, prepended to raw key bytes):

| Key | Prefix | Total DER len |
|---|---|---|
| Ed25519 private (PKCS#8) | `302e020100300506032b657004220420` | 48 |
| Ed25519 public (SPKI) | `302a300506032b6570032100` | 44 |
| ECDSA private (PKCS#8-style) | `3030020100300706052b8104000a04220420` | 50 |
| ECDSA private (SEC1 alt) | `30540201010420` | 39 |
| ECDSA public (standard SPKI, compressed) | `3036301006072a8648ce3d020106052b8104000a032200` | 56 |
| ECDSA public (legacy) | `302d300706052b8104000a032200` | 47 |

Raw lengths: Ed25519 private seed 32 (some libs use 64 = seed‖pubkey), Ed25519
public 32; ECDSA private scalar 32, ECDSA public 33 compressed (`02/03‖X`) or 65
uncompressed (`04‖X‖Y`).

`from_bytes` dispatches on length (32→raw, known DER totals→DER); `from_string`
strips an optional `0x` then hex-decodes. Always ALSO provide explicit
`from_string_ed25519` / `from_string_ecdsa` / `from_string_der` variants.

Trap: the JS SDK's ECDSA `toBytesDer()` emits the **legacy** 47-byte form while
parsing both — interop requires accepting both on input.

## 2. Signing

- **What is signed**: the raw `bodyBytes` of each per-node SignedTransaction.
  No transaction-layer prehash for Ed25519.
- **Ed25519**: standard detached signature (internally SHA-512). Output 64 bytes.
- **ECDSA secp256k1**: hash the message with **keccak256** (NOT sha256), sign the
  32-byte digest, emit **compact 64-byte `r‖s`** (NOT DER). Beware library labels:
  the Python SDK passes `Prehashed(SHA256())` purely to declare digest length — the
  digest is keccak256. Verification should accept both compact and DER input.
- **SignaturePair prefixes**: Ed25519 uses the full 32-byte raw public key as
  `pubKeyPrefix`; ECDSA uses the 33-byte compressed key.
- Recovery IDs (for EVM interop) are optional; JS derives them by scanning 0–3.

## 3. EVM address derivation (ECDSA only)

`keccak256(uncompressed_pubkey[1:])[-20:]` — uncompressed 65-byte key, drop the
`04` prefix, keccak the 64 bytes, take the last 20. Reject the operation for
Ed25519 keys. Wrap in an EvmAddress type enforcing 20 bytes / 40 hex chars.

## 4. Mnemonics and HD derivation

BIP-39 (12 or 24 words; 22-word legacy format with crc8 checksum exists for
recovery only). The standard derivations every SDK should implement:

- **Ed25519**: BIP-39 seed → SLIP-10 with HMAC key `"ed25519 seed"` → path
  `m/44'/3030'/0'/0'/index'` (SLIP-10 ed25519 hardens every step). This is
  `toStandardEd25519PrivateKey` — the cross-SDK/wallet-compatible derivation.
- **ECDSA**: BIP-39 seed → BIP-32 with HMAC key `"Bitcoin seed"` → path
  `m/44'/3030'/0'/0/index` (first three hardened). Coin type 3030 = Hedera;
  wallets may also use the ETH path `m/44'/60'/0'/0/index`.
- Legacy derivations (PBKDF2-based `legacyDerive`, 2048 iterations, salt `0xff`)
  are kept only for recovering old wallets.

**Never invent a derivation** (e.g. seed[:32] as a key): deterministic-but-
nonstandard derivation produces mnemonics that recover nothing in any other tool —
the worst silent failure a crypto API can have. Acceptance criterion for any
mnemonic implementation: known-answer vectors from the JS/Java SDK test suites.

## 5. Round-trip matrix and the 32-byte trap

Provide the full matrix per key type: `to/from_bytes_raw`, `to/from_bytes_der`,
`to/from_string_raw`, `to/from_string_der`, plus generic `to_bytes`/`to_string`.

Cross-SDK divergence to be aware of: JS `PrivateKey.toString()` returns **DER hex**
while Python `to_string()` returns **raw hex**; JS `toBytes()` is raw for Ed25519
but DER for ECDSA. When porting, pick one default and document it loudly.

**The 32-byte ambiguity trap**: a raw Ed25519 private seed, a raw Ed25519 public
key, and a raw ECDSA scalar are all 32 indistinguishable bytes. Generic
`from_bytes`/`from_string` must pick a documented default (both SDKs default to
Ed25519 private) and should emit a warning steering users to the explicit typed
constructors — Python does; copy that.

## 6. PEM / keystore

JS supports PEM (`BEGIN EC PRIVATE KEY` → ECDSA branch; PKCS#8
EncryptedPrivateKeyInfo for passphrase-protected Ed25519; AES-128-CBC for legacy
EC PEM) and a JSON keystore (`fromKeystore`/`toKeystore`). Keystores persist only
raw key bytes — restored keys lose the chain code and cannot derive children;
document that. Python currently supports neither (unencrypted DER only) — a known
gap, not a convention.

## 7. Porter gotchas checklist

- [ ] ECDSA signs keccak256, emits compact r‖s (not sha256, not DER)
- [ ] DER prefixes match the table exactly; accept legacy + standard ECDSA SPKI
- [ ] SignaturePair prefix = full raw key (Ed25519) / compressed key (ECDSA)
- [ ] EVM address = keccak(uncompressed[1:])[-20:], ECDSA only
- [ ] Standard mnemonic paths (`m/44'/3030'/0'/0'/i'` SLIP-10 / `.../0/i` BIP-32)
      verified against cross-SDK known-answer vectors
- [ ] Generic 32-byte input: documented default + warning + explicit typed variants
- [ ] Default `to_string`/`to_bytes` format documented (raw vs DER differs by SDK)
