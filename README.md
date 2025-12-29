![EasyKey Architecture](docs/diagrams/easykey_architecture.svg)

# EasyKey Wallet Architecture & Security Design

Implementation-focused technical documentation for a React Native Web3 wallet that isolates secrets in native code and a Rust core. This repository is intended for architecture review, security discussion, and long-term documentation of design decisions.

> This is **not** a formal security audit and must not be treated as a security guarantee. It describes the intended architecture and (where applicable) the current implementation model.

## What this repo contains

- A **technical whitepaper** describing the end-to-end architecture and security boundaries:
  - React Native (JS/TS) orchestration and non-sensitive state
  - Native module boundary (`@easykey/secure-crypto-native`) for secure input and master key handling
  - Rust `wallet_core` for derivation, signing, and crypto helpers
- Data storage model and payload formats (`ekws1:*`, `ekvault1:*`)
- Threat model assumptions, operational security notes, and known limitations

## Read the whitepaper

- English: `docs/TECHNICAL_WHITEPAPER.md`
- Diagram (source): `docs/diagrams/easykey_architecture.svg`

Suggested reading order: **System overview → Data storage model → Key formats/payloads → Signing model → Destruction logic**.

## Design principles (high level)

### 1) Secrets stay out of JavaScript
In the “native isolation” path, JavaScript should never receive plaintext:
- mnemonic / private keys
- master keys
- decrypted vault text

JS is limited to:
- orchestration (which native call to invoke)
- UI/navigation
- storing **non-sensitive metadata** (wallet index, labels, cached state)

### 2) Native boundary uses opaque handles and encrypted payloads
- A master key is derived in native code and referenced via an **opaque handle** (e.g., `mkHandle`), rather than exposing key bytes to JS.
- Per-wallet/per-chain secrets are stored in Keychain/Keystore as **ciphertext strings** and are decrypted only within native/Rust.

### 3) Rust core owns correctness-critical crypto and parsing
Rust is used for:
- derivation (BIP39/BIP44/SLIP-10 variants as applicable)
- signing for supported chains
- transaction parsing / summaries used in trusted confirmation UX
- vault encryption/decryption helpers
- in-memory secret/session stores to avoid long-lived managed strings

## Threat model (what this aims to defend against)

This design primarily targets:
- Offline compromise of app storage (AsyncStorage copies, device backups, filesystem extraction)
- UI-level attacks that rely on secrets passing through JS (logging, instrumentation, accidental persistence)
- “Bridge-level” tampering where JS tries to modify signing/reveal data without native/Rust verification

This design does **not** fully solve:
- A fully compromised device/OS (kernel/root/jailbreak)
- Malicious accessibility overlays or hardware implants
- Side-channel attacks beyond the scope of typical mobile hardening
- Vulnerabilities in external dependencies, RPC providers, or chain infrastructure

## Non-goals (explicit)

- This repository is not a certification, audit report, or formal proof of security.
- We do not claim that “no secret can ever leak.” The goal is to **reduce attack surface** and apply defense-in-depth.
- We do not claim GPU/ASIC “impossibility” for password guessing. Cost factors are engineering tradeoffs.

## What we want reviewed

If you are reviewing the architecture, the most valuable feedback usually falls into these categories:

1. **Handle lifecycle safety**
   - Disposal correctness
   - Concurrency/threading pitfalls
   - FFI boundary mistakes that could leak secrets

2. **In-memory secret/reference stores**
   - Lifetime hazards
   - Crash/restart behavior
   - Risk of accidental persistence via logs, debug builds, or managed strings

3. **Payload formats and versioning**
   - Forward/backward compatibility
   - Migration strategy and “format agility”
   - Adequacy of domain separation, parameter binding, and integrity checks

4. **Trusted UX**
   - Whether confirmation modals meaningfully reduce JS tampering risk
   - Secure input design to prevent plaintext from entering JS event streams
   - Screen capture/app switcher snapshot mitigations (realistic constraints)

## Repository structure (expected)

> This repo is documentation-first. If you link to code repositories or submodules, keep the whitepaper grounded in real paths and commit hashes where possible.

```
docs/
  TECHNICAL_WHITEPAPER.md
  diagrams/
    easykey_architecture.svg
```

## Responsible disclosure

If you believe you found a security issue that could put users at risk, please avoid public disclosure until it is triaged. Please do not post exploit details in public issues.

- Open a GitHub Security Advisory (preferred), or
- Open a private issue with minimal details and request a secure channel for reproduction steps.

## License

Unless otherwise specified, documentation in this repository is provided under the repository’s license (see `LICENSE`).

## Disclaimer

This documentation is provided “as is” without warranty of any kind. Use it for review and discussion, not as a substitute for professional security auditing.
