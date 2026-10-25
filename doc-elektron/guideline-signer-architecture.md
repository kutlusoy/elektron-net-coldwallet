# Elektron Net: `elektron-net-coldwallet` Signer Architecture Guideline

- **Version:** 0.1 (draft)
- **Date:** October 25, 2026
- **Audience:** Firmware developers, security reviewers, wallet integrators, release engineers
- **Authors:** Elektron Net Developers, Ali Kutlusoy ([@kutlusoy](https://x.com/kutlusoy))
- **Reference implementation:** [`elektron-net`](https://github.com/kutlusoy/elektron-net) `src/kernel/chainparams.cpp`, `src/wallet/walletutil.cpp` (`GenerateWalletDescriptor()`, `ELEK_COIN_TYPE`), `doc/psbt.md` (treat as ground truth for chain parameters, derivation paths and PSBT handling)
- **Fork base:** [`SeedSigner/seedsigner`](https://github.com/SeedSigner/seedsigner) (MIT)
- **Consumer:** [`elektron-net-mobile`](https://github.com/kutlusoy/elektron-net-mobile) (watch-only coordinator and broadcaster)
- **See also:** [`guideline-wallet-integration.md`](https://github.com/kutlusoy/elektron-net/blob/main/doc-elektron/guideline-wallet-integration.md), [`CHANGELOG-slip44-coin-type.md`](https://github.com/kutlusoy/elektron-net/blob/main/doc-elektron/CHANGELOG-slip44-coin-type.md), [`guideline-escrow-psbt-and-tools.md`](https://github.com/kutlusoy/elektron-net-mobile/blob/main/doc-elektron/guideline-escrow-psbt-and-tools.md), [`guideline-entropy-and-randomness.md`](https://github.com/kutlusoy/elektron-net-mobile/blob/main/doc-elektron/guideline-entropy-and-randomness.md), [`WHITEPAPER.md`](https://github.com/kutlusoy/elektron-net/blob/main/WHITEPAPER.md)
- **Website:** https://elektron-net.org | **X:** @elektronnet, @kutlusoy | **Telegram:** @elektronnet

Requirement levels follow common practice: **MUST** = mandatory for correct/safe operation, **SHOULD** = strongly recommended, **MAY** = optional.

---

## 1. Scope and Decisions

`elektron-net-coldwallet` is an air-gapped signing device for Elektron Net (ELEK). It generates and holds the private keys, shows receive addresses and the account descriptor, and signs transactions. All communication with the outside world uses QR codes, plus a microSD card for firmware images.

Decisions of October 25, 2026:

| # | Decision |
|---|---|
| D1 | Reference platform: a fork of SeedSigner on a Raspberry Pi Zero v1.3 (no radio hardware) with a camera and a 240x240 display |
| D2 | Data channel: QR codes only. No USB data, no Bluetooth, no Wi-Fi, no network stack |
| D3 | Firmware updates: signed image on a microSD card, flashed on a separate computer |
| D4 | Derivation: SLIP-44 coin type `1370'` only. The legacy `0'` path is not supported |
| D5 | The device holds keys in two modes: persistent (encrypted on the SD card) and stateless (seed entered per session) |
| D6 | Repository language: English |
| D7 | Hardware-wallet compatibility with the node wallet (`elektrond`) and `elektron-net-mobile` is a design requirement from the first release |

Non-goals of v1: Lightning, Bluetooth or NFC transport, smart-card or secure-element support, closed-source components, support for other chains.

## 2. Threat Model

| Threat | In scope | Mitigation |
|---|---|---|
| Malware on the online phone or PC | Yes | Keys never leave the device. The device shows recipient, amount and fee on its own display before signing |
| Remote attack on the signer | Yes | No radio, no USB data lines in use, QR as the only input |
| Weak or backdoored randomness | Yes | Hardware RNG, camera and dice entropy are mixed, health tests run, no fallback to weak sources (§4.1) |
| Theft of the device or SD card | Yes | Seed encrypted with a passphrase-derived key (§4.2), stateless mode available |
| Tampered firmware image | Yes | Signed images, hash shown on screen, reproducible build (§9) |
| Evil maid with physical access and hardware tools | Partly | The Pi Zero has no secure element and no verified boot. Tamper evidence and stateless mode reduce the risk, they do not remove it |
| Side channels and fault injection against the SoC | No | Out of scope without a secure element |

Stated plainly: this design gives an air gap, an open and auditable code base and a trusted display. It does not give tamper resistance against a well-equipped attacker holding the device.

## 3. Hardware

| Part | Requirement |
|---|---|
| Board | Raspberry Pi Zero v1.3 (no Wi-Fi or Bluetooth hardware). A Pi Zero W or 2 W has a radio that cannot be removed and is not recommended |
| Display | 240x240 pixel LCD with buttons (the model used by SeedSigner). Other Waveshare boards with the same look are not compatible |
| Camera | Pi Zero compatible camera module |
| Storage | microSD card |
| Power | Battery or a power-only cable. A data-capable USB connection to a computer MUST NOT be used while keys are loaded |
| Case | Open design, tamper-evident seals RECOMMENDED |

Availability of the Pi Zero v1.3 is limited. Alternative boards are only acceptable if they meet the same requirements: no radio hardware, a hardware RNG, a camera and a display. This list is to be extended after evaluation.

## 4. Key Management

### 4.1 Seed generation

- **MUST:** at least 128 bits of entropy (12 words), 256 bits (24 words) SHOULD be the default.
- **MUST:** the seed is derived from a hash (HKDF-SHA256) over all available sources: the SoC hardware RNG (`/dev/hwrng`), camera sensor noise, and, optionally, user dice rolls. A single source failing its health test aborts generation.
- **MUST:** health tests on the hardware RNG at every start (repetition count test and adaptive proportion test as in NIST SP 800-90B). There is no fallback to a weaker source.
- **SHOULD:** a dice mode as in `elektron-net-mobile` (50 rolls for 12 words), so the seed can be recomputed without trusting the chip.
- **MUST:** BIP-39 mnemonic, English word list, so the same words can be restored in `elektron-net-mobile` and any BIP-39 compatible tool. Using the same seed on the cold wallet and on a hot wallet defeats the purpose, and the UI SHOULD warn against it.

### 4.2 Persistent storage

- **MUST:** the seed is stored only encrypted. The key is derived from a passphrase with a memory-hard function (Argon2id, or scrypt with comparable cost), tuned for several seconds on the Pi Zero.
- **MUST:** a minimum passphrase length is enforced. A short numeric PIN alone is not accepted, because an attacker holding the SD card can brute force offline and the Pi Zero has no secure element with an attempt counter.
- **MUST:** the encrypted blob lives on a data partition of the SD card, the system partition is read-only at runtime.
- **MUST:** keys exist in RAM only while needed, and are overwritten before power off or return to the main menu.
- **SHOULD:** the optional BIP-39 passphrase (25th word) is never stored.

### 4.3 Stateless mode

The seed is entered per session by SeedQR or mnemonic and is never written to disk. This mode SHOULD be the recommended default for high-value cold storage.

### 4.4 Backup

- The mnemonic is shown on the device, once, with a verification step. It is written on paper or steel.
- A SeedQR for fast restore MAY be shown. Printing or saving a SeedQR on a networked computer is discouraged in the UI.
- The device writes no backup file anywhere.

## 5. Chain Integration

### 5.1 Network parameters (mainnet, from `chainparams.cpp`)

| Parameter | Value |
|---|---|
| Bech32 HRP | `be` (`be1q...`, Taproot `be1p...`) |
| Base58 P2PKH / P2SH / WIF | `0x00` / `0x05` / `0x80` (identical to Bitcoin) |
| Extended keys | `0488B21E` / `0488ADE4` (identical to Bitcoin) |
| SLIP-44 coin type | `1370` (`ELEK`) |
| Testnet / Signet | HRP `tb`, coin type `1'`, Base58 `111`/`196`/`239` |
| Regtest | HRP `bcrt`, coin type `1'` |
| Unit | 1 ELEK = 100,000,000 lepton |

Because xpub and WIF encodings are byte-identical to Bitcoin, the device cannot recognise the chain from key material. The firmware therefore:

- **MUST** be built for exactly one chain. There is no network selector for Bitcoin.
- **MUST** show the symbol ELEK and the HRP `be` on every screen that confirms an address, an amount or a descriptor.
- **MUST** use `lepton` and `ELEK` for amounts, never `sats` or `BTC`.
- **MUST** refuse PSBTs whose outputs use a different HRP.
- A separate testnet build SHOULD exist. The HRP `tb` collides with Bitcoin testnet, so the build MUST be visibly marked as testnet on every screen.

### 5.2 Derivation paths

| Purpose | Path | Notes |
|---|---|---|
| Native SegWit single-sig (default) | `m/84'/1370'/0'` | `wpkh()`, receive `/0/*`, change `/1/*` |
| Taproot single-sig | `m/86'/1370'/0'` | `tr()`, SHOULD be supported |
| Nested SegWit, legacy | `m/49'/1370'`, `m/44'/1370'` | MAY be supported, discouraged by the UI |
| Multisig / escrow key | `m/48'/1370'/<a>'/2'` | P2WSH, identical to `elektron-net-mobile` escrow (§2.1 of its guideline) |

- **MUST:** coin type `1370'` on mainnet. A PSBT that carries key origin paths with another coin type for the user's own inputs or change MUST be rejected with an explicit message.
- The legacy `0'` path is not supported, which is safe because the device only holds seeds created on it.

### 5.3 Changes to the SeedSigner code base

- Add an Elektron Net network definition to the underlying key library (HRP, version bytes, coin type) and set it as the only mainnet.
- Adapt the change-output check, which compares derivation paths against the coin type, to `1370'`.
- Replace currency strings and fee display (ELEK, lepton, lep/vB).
- Remove or hide any feature that depends on an online service.
- Keep all changes small and rebase-friendly, and keep upstream license and notices unchanged. Add "Elektron Net Developers" as additional attribution where the MIT license allows.

## 6. QR Transport

| Direction | Content | Format |
|---|---|---|
| Device to wallet | Account descriptor with key origin, `[fp/84h/1370h/0h]xpub.../<0;1>/*` | Static QR, plain text descriptor |
| Device to wallet | Receive address | Static QR, `elek:` payment URI SHOULD be supported |
| Wallet to device | Unsigned PSBT | Static QR if it fits, otherwise animated QR |
| Device to wallet | Signed PSBT | Animated QR |
| Wallet to device | Message to sign | Static or animated QR |

- **MUST:** support both the single Base64 PSBT QR (up to about 1,800 characters, as in `elektron-net-mobile` today) and an animated multi-part format. The animated format is to be fixed jointly with `elektron-net-mobile` (§8).
- **MUST:** the device never accepts anything but PSBTs, descriptors, messages and seeds through the camera. Unknown QR content is ignored with a visible message.
- **MUST:** no PSBT bytes are written to disk.

## 7. Signing Flow and On-Device Verification

```text
Mobile (watch-only):  walletcreatefundedpsbt  ->  PSBT shown as QR
Signer:               scan  ->  parse  ->  verify  ->  confirm  ->  sign  ->  show signed PSBT as QR
Mobile:               scan  ->  finalizepsbt  ->  sendrawtransaction
```

The device **MUST** display, before any signature, for the exact file that is signed:

1. Every output with its full address (in pages if needed) and its amount in ELEK.
2. Which outputs are own change, recognised by derivation path and key origin, and which are external.
3. The total fee in ELEK and the fee rate in lepton per vB. A warning is shown when the fee exceeds 5 percent of the sent amount or the rate exceeds 1000 lep/vB, the same thresholds as `elektron-net-mobile`.
4. The number of inputs and, for multisig, the number of signatures already present.
5. The master fingerprint of the signing key.

The device **MUST** reject a PSBT when: an output address is not valid for HRP `be`, the sum of inputs minus outputs does not match the stated fee, an input amount cannot be determined, or the key origin of an own input does not match the loaded seed.

**Pruned network consideration:** Elektron Net nodes keep only about 137 days of blocks. A previous full transaction (`non_witness_utxo`) may therefore not be obtainable for older coins. The firmware **MUST** accept SegWit inputs that carry only `witness_utxo`, and **MUST** verify amounts from it. Taproot inputs commit to all input amounts in the signature hash and need no previous transaction.

## 8. Compatibility

### 8.1 Node wallet (`elektrond`)

| Need | Node feature | Status |
|---|---|---|
| Create unsigned PSBT | `walletcreatefundedpsbt` | Available |
| Add UTXO data | `utxoupdatepsbt`, `walletprocesspsbt` (no signing) | Available, SegWit inputs only |
| Watch-only wallet from device xpub | `importdescriptors` into a wallet with private keys disabled | Available |
| Inspect before broadcast | `decodepsbt`, `analyzepsbt` | Available |
| Combine, finalize, broadcast | `combinepsbt`, `finalizepsbt`, `sendrawtransaction` | Available |
| Derivation | `GenerateWalletDescriptor()` uses `1370h` on mainnet | Matches |

No change to the node or to consensus is needed.

### 8.2 `elektron-net-mobile`

Present today: PSBT check, sign, combine, finalize and broadcast (`PsbtTools.kt`), single QR scan and display up to about 1,800 characters, multisig escrow with BIP-48 paths, public descriptor export, message signing.

Gaps to close in `elektron-net-mobile` (tracked there, listed here as dependencies):

| # | Gap | Reference |
|---|---|---|
| G1 | Animated QR encode and decode for PSBTs | Escrow guideline E2 (Phase 4) |
| G2 | Watch-only wallet type created from a scanned device descriptor | Not in the feature list yet |
| G3 | Send flow for watch-only wallets: show PSBT as QR, scan the signed PSBT, finalize, broadcast | Building blocks exist in `PsbtTools.kt` |
| G4 | Hardware and air-gapped signer as escrow participant | Escrow guideline E3 (Phase 5) |
| G5 | Interoperability tests with an external signer | Escrow guideline checklist |

Both repositories MUST agree on one animated QR format before either implements it. Candidates are the UR2 `crypto-psbt` format and BBQr. The choice is an open question (§12).

### 8.3 `elektron-net-electrum`

The Electrum fork keeps `m/0'` for its own native seeds and is not compatible with the `1370'` rule of this device. It can still import the device descriptor as a watch-only wallet and exchange PSBTs, which makes it a possible first offline-signing path before `elektron-net-mobile` supports watch-only wallets. This needs verification.

## 9. Firmware Distribution and Updates

- **MUST:** release images are built reproducibly from a tagged commit. The build manifest lists the commit hash, the base OS image hash and the SHA-256 of the final image.
- **MUST:** each release image has a detached signature made with the project release key. The public key and its fingerprint are published in this repository and on https://elektron-net.org.
- **MUST:** an update is done by flashing the signed image to the microSD card on a separate computer. The device never updates itself and has no code path that reads executable content from a card it did not boot from.
- **MUST:** the device shows its firmware version and image hash at start, and in the menu.
- **MUST:** after an update the user restores the seed from the written backup. The persistent data partition is not migrated between images in v1.
- **SHOULD:** a documented verification procedure for users (hash and signature check) ships with every release.
- Verified boot is not available on the Pi Zero. This limitation MUST be stated in the user documentation.

## 10. Security Requirements

| # | Requirement | Level |
|---|---|---|
| C1 | No radio hardware, no network stack, no USB data use while keys are loaded | MUST |
| C2 | Entropy from several sources, hashed together, with health tests and no weak fallback | MUST |
| C3 | Seed stored only encrypted, with a memory-hard KDF and a minimum passphrase length | MUST |
| C4 | Keys are held in RAM only while needed and overwritten afterwards | MUST |
| C5 | Every signature is preceded by the full on-device verification screen (§7) | MUST |
| C6 | Chain, symbol and HRP are fixed in the build and shown on confirmation screens | MUST |
| C7 | Coin type `1370'` only, other coin types for own keys are rejected | MUST |
| C8 | `witness_utxo` only SegWit inputs are accepted and amounts verified | MUST |
| C9 | Signed images, reproducible build, hash shown on screen | MUST |
| C10 | No logging of seeds, keys, PSBTs or addresses to disk | MUST |
| C11 | Stateless mode available and recommended for high-value storage | SHOULD |
| C12 | Independent security review before the first stable release | MUST |

## 11. Test Plan

- **Unit tests:** network definition (addresses, HRP, WIF), derivation vectors for `84'/1370'/0'`, `86'/1370'/0'` and `48'/1370'/a'/2'`, entropy health tests with injected faults.
- **Interoperability against a regtest `elektrond`:** create a watch-only wallet from the device descriptor, fund it, create a PSBT with `walletcreatefundedpsbt`, sign on the device (simulator and real hardware), finalize and broadcast. Cover SegWit v0 inputs with and without `non_witness_utxo`, Taproot inputs, change detection, multi-input PSBTs that need animated QR.
- **Negative tests:** wrong HRP, wrong coin type, fee mismatch, missing amounts, malformed QR, oversized PSBT.
- **Mobile:** end-to-end send from `elektron-net-mobile` once gaps G1 to G3 are closed.
- **Hardware:** radio absence check, power analysis of boot, and the verification procedure for images.

## 12. Open Questions

| # | Question | Recommendation |
|---|---|---|
| Q1 | Animated QR format: UR2 `crypto-psbt` or BBQr? | Decide together with `elektron-net-mobile` before Phase 1 ends. Pick the format the SeedSigner code base already supports, to keep the fork small |
| Q2 | Does upstream SeedSigner accept `witness_utxo` only SegWit v0 PSBTs? | Verify in the code. If not, add support (C8) |
| Q3 | Replacement for the Pi Zero v1.3 when it is no longer available | Evaluate boards with the same properties. A radio that is merely disabled in software is a weaker option |
| Q4 | Minimum passphrase length and KDF cost | Define after benchmarks on the Pi Zero |
| Q5 | Testnet HRP collision with Bitcoin testnet | Assign a dedicated testnet HRP in the node, then update this device |
| Q6 | Offer a smart-card or secure-element upgrade path later | Not for v1 |

## 13. Checklist

- [ ] Fork SeedSigner and add the Elektron Net network definition
- [ ] Derivation paths and change detection for coin type `1370'`
- [ ] Persistent encrypted seed storage and stateless mode
- [ ] Entropy mixing with hardware RNG, camera, dice, and health tests
- [ ] Animated QR format agreed with `elektron-net-mobile`
- [ ] On-device verification screen (§7)
- [ ] Signed, reproducible release images and user verification guide
- [ ] Regtest interoperability tests with `elektrond`
- [ ] `elektron-net-mobile` gaps G1 to G3 closed
- [ ] Independent security review
