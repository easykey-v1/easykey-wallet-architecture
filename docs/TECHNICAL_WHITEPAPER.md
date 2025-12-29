# EasyKey Wallet Technical Whitepaper (Public Draft)
**Architecture & Security Design — for discussion and review**

> **Status (Public Draft):** This document is published to solicit architecture/security feedback on a React Native Web3 wallet design that isolates secrets in native code and a Rust core. The production codebase is currently private (pre‑commercial).  
> **Important:** This is **not** a formal security audit and must not be treated as a security guarantee. It describes the intended architecture and (where applicable) the current implementation model.

## 0) Public release notes

To make Reddit/OSS discussion productive without publishing the full product code, this public draft:
- **Includes:** system architecture, trust boundaries, data flow, storage model, payload format versioning, threat model, and key review questions.
- **Intentionally redacts:** exact Keychain/Keystore item names, internal package/file paths, and exact anti‑brute‑force timing constants. These details do not materially change the design discussion but can provide unnecessary attacker “fingerprints”.

Planned transparency roadmap (high level):
- Phase 1 (now): publish architecture + formats for review.
- Phase 2: open-source a **minimal SDK** / reference library for payload formats + boundary-safe primitives.
- Phase 3: selectively open additional components (e.g., Rust crypto helpers), while keeping the full consumer app private if needed.

---

## 1) System overview

High‑level architecture (data flow):

```
React Native (JS/TS)
  ├─ UI + navigation
  ├─ Orchestration (which native call to make)
  ├─ Stores non-sensitive metadata in AsyncStorage
  └─ Designed so that JS does not need plaintext mnemonic/private key in the “native isolation” path

        │
        ▼
Native security boundary (iOS/Android)
  ├─ Secure input views (plaintext stays in native UI; not surfaced via JS text events)
  ├─ Short-lived in-memory references (e.g., PIN ref)
  ├─ libsodium-backed primitives (Argon2id, XChaCha20-Poly1305, etc.)
  ├─ MasterKey handles (native memory; JS holds only an integer handle)
  ├─ Biometric wrapper (Keychain/Keystore-protected masterKey blob)
  └─ Bridges to Rust wallet_core for signing / chat crypto / vault crypto

        │
        ▼
Rust `wallet_core` (iOS static lib + Android .so)
  ├─ Wallet derivation (BIP39/BIP44, Solana SLIP-10, etc.)
  ├─ Custom phrase validation (entropy + pattern rules) and deterministic derivation
  ├─ SecretStore (ephemeral in-memory secret references, `eksecret1:*`)
  ├─ SessionStore (derivation sessions)
  ├─ Signing (EVM / BTC PSBT / Solana / Tron)
  ├─ Chat crypto helpers
  └─ Vault encryption/decryption (ekvault1:* payload)
```

Primary design goal:
- **Secrets do not cross into JS** in the secure (“native isolation”) path; JS orchestrates and renders UI, while sensitive operations occur in native/Rust.

---

## 2) Components and responsibilities

### 2.1 React Native (JS/TS)
**Responsible for:**
- UI/UX, navigation, state management, networking (RPC).
- Storing **non-sensitive** indexes/caches (wallet list, labels, app settings, cached state).
- Orchestrating calls into native/Rust for anything that touches secrets.

**Not responsible for:**
- Persisting wallet password, mnemonic, private keys, or master keys.
- Performing signing with JS-held private keys in the secure path.

### 2.2 Native security boundary (iOS/Android module)
This is the security boundary that:
- Implements **secure input** so user-typed plaintext is not emitted to JS text events.
- Derives a **MasterKey** from the wallet password and exposes it to JS only via an **opaque handle** (e.g., `mkHandle`), never as raw bytes.
- Stores encrypted secrets in **Keychain/Keystore** and supports **biometric wrapping** to recreate ephemeral handles without frequent password entry.
- Provides **trusted confirmation / reveal UX** (native modals) so critical transaction summaries and sensitive displays do not rely solely on JS rendering.

### 2.3 Rust `wallet_core`
Rust is used where correctness, parsing, and isolation matter:
- Address derivation, validation, signing, and transaction parsing/summaries.
- Payload formats for vault and secret storage.
- In-memory secret/session stores to avoid long-lived managed-language `String` lifetimes for plaintext.

---

## 3) Data storage model (what is stored where)

### 3.1 AsyncStorage (JS-accessible, not secret-safe)
Used for **indexes and caches** only (treat as copyable/offline-readable):
- Wallet index (addresses, labels, non-sensitive metadata)
- App state and feature flags (lock enabled, biometric signing enabled, etc.)
- Non-secret cached UI state

**Rule:** AsyncStorage must not contain mnemonic/private keys/master keys or decrypted vault contents.

### 3.2 Keychain / Keystore (OS protected)
Used for **encrypted wallet secrets** and security-critical records.

Encrypted per-wallet/per-chain secret entry:
- Stored under app-defined Keychain/Keystore records (exact item keys redacted in this public draft).
- Payload format (ciphertext string): `ekws1:<nonceHex>:<cipherB64>`

Optional “sentinel” entry:
- A separate record used to probe existence/sync without forcing biometric/passcode prompts.

Security records (examples):
- Wallet password verification record (salt/params + verification hash)
- Destroy/duress PIN record (hash only; never plaintext)

### 3.3 Native in-memory refs (ephemeral)
A native in-memory store holds short-lived sensitive references (e.g., a PIN reference) that can be consumed to derive a MasterKey handle.

### 3.4 Rust in-memory SecretStore / SessionStore
`wallet_core` maintains:
- **SecretStore:** stores normalized secrets in Rust memory and returns an opaque `eksecret1:*` reference (JS never receives plaintext).
- **SessionStore:** stores derivation sessions and returns a session reference (JS receives only derived addresses/summary data).

---

## 4) Core cryptographic primitives and payload formats

### 4.1 Master key derivation (wallet password → master key handle)
- KDF: **Argon2id** (libsodium-compatible)
- Output: opaque handle (e.g., `mkHandle`) referencing key bytes held in native memory
- JS never receives the key bytes

Parameters:
- Wallet unlock KDF uses a performance/safety tradeoff (moderate memory) suitable for frequent unlock on mobile.
- Stronger KDF parameters may be used for vault operations (see 4.3).

### 4.2 Wallet secret encryption payload (ekws1)
- AEAD: **XChaCha20‑Poly1305 (IETF)**
- Format: `ekws1:<nonceHex>:<cipherB64>`
- Stored in Keychain/Keystore as ciphertext; decrypted only inside native/Rust using an active master key handle.

### 4.3 Vault encryption payload (ekvault1)
Vault “encrypt/decrypt arbitrary text” uses:
- KDF: Argon2id (configurable; may use higher memory settings)
- AEAD: XChaCha20‑Poly1305
- Example format: `ekvault1:v2:<ops>:<memBytes>:<saltHex>:<nonceHex>:<cipherB64>`

Operational note:
- Clipboard hygiene for “copy” actions is best-effort (auto-clear after a short window if not overwritten).

---

## 5) Wallet password, unlock gate, and master key lifecycle

### 5.1 Setup
- Wallet password/PIN is collected via secure native UI.
- Native derives a MasterKey and stores a verification record (salt/params + verification hash).
- JS receives only an opaque handle during the session; it must be disposed after use.

### 5.2 Unlock
- User input is verified in native code against the stored verification record.
- On success, native returns an ephemeral handle; JS uses it only to authorize operations and then disposes it.

### 5.3 Session lock gate
- “App lock enabled” is stored as a non-sensitive flag.
- “Session unlocked” is maintained in memory only.

---

## 6) Biometrics + wallet password (combined model)

Biometrics is **not a replacement password**; it is a convenience layer to:
- Unlock an OS-protected blob and recreate an ephemeral MasterKey handle.
- Reduce repeated password entry for sensitive flows (when enabled).

Enrollment requires a wallet-password-derived handle. If biometric enrollment changes or is invalidated by the OS, biometric mode is disabled and the system falls back to password.

---

## 7) Create/import flows (native isolation path)

### 7.1 Secure inputs
Secure input views support multiple validators (e.g., BIP39 mnemonic, private key, custom phrase, raw text). Validation runs in native/Rust and stores plaintext in Rust SecretStore, returning only:
- `referenceId` (opaque `eksecret1:*`)
- validation metadata (valid/reason/token count/entropy estimate), without returning actual secret material to JS

### 7.2 Derivation sessions
From `referenceId`, JS calls into Rust to derive addresses/keys. Rust stores a session in SessionStore and returns:
- session reference id
- derived address summary (addresses only)
- optional custom-phrase metadata (e.g., version + short key id)

### 7.3 Finalization
After the user unlocks (obtains a master key handle), Rust/native encrypts per-chain secrets into `ekws1:*` payloads which are stored to Keychain/Keystore. JS stores only ciphertext and updates AsyncStorage indexes.

### 7.4 Trusted reveal
When the user chooses to “reveal mnemonic/private key”, the preferred design is:
- JS loads ciphertext from Keychain/Keystore + obtains handle
- Native/Rust decrypts and renders in a native modal
- Plaintext is not returned to JS

---

## 8) Custom phrase (“EasyKey phrase”) rules and derivation

### 8.1 Validation
Validation is a gate before derivation. The rules aim to reject low-entropy, pattern-like phrases by combining:
- minimum length requirements
- numeric-run constraints and pattern blocklists
- diversity requirements across character categories
- entropy estimation with penalties for repeats and predictable structures

### 8.2 Derivation
Deterministic derivation uses:
- Argon2id (higher-cost profile) to derive a root key from the validated phrase
- HKDF-SHA256 with domain separation to derive per-chain keys

Design goal:
- Strong offline guessing resistance even if the phrase is weaker than desired, while recognizing this is a tradeoff for low-memory devices.

---

## 9) Signing model (EVM / BTC / TRON / SOL)

Common pattern:
1) JS loads encrypted key payload (`ekws1:*`) from Keychain/Keystore.
2) JS obtains an ephemeral master key handle (biometric or password).
3) JS passes `(payload, tx/message, handle)` to native signing.
4) Native/Rust decrypts payload and signs; Rust may parse/validate and produce trusted summaries for confirmation UI.
5) Dispose handle.

Trusted confirmation UX:
- For transactions, the confirmation modal uses fields **parsed/decoded in Rust** rather than relying on JS-only summaries.

---

## 10) Encrypted chat transport (design notes)

If using a blockchain as message transport:
- Messages are end-to-end encrypted using asymmetric key agreement + symmetric AEAD/secretbox.
- Wallet private keys remain encrypted at rest and are decrypted only inside native/Rust using a master key handle.
- The on-chain payload contains ciphertext + nonce; plaintext is never emitted to JS.

(Exact chain encoding details are out of scope for this public draft.)

---

## 11) Destruction / duress model (high level)

Two independent mechanisms may exist:

1) **Progressive anti‑brute‑force + auto-destroy**
- Failed unlock attempts are tracked in an OS-protected record.
- Cooldown delays increase after repeated failures.
- After a threshold, the wallet secrets are destroyed (Keychain/Keystore entries removed and indexes cleared).

2) **Duress / destroy PIN**
- A separate PIN triggers destruction when entered at unlock.
- Stored as a hash only; never plaintext.
- Optional “decoy wallet” mode can preserve a minimal profile while destroying real secrets.

(Exact numeric schedules are intentionally omitted from this public draft and can be discussed abstractly.)

---

## 12) Threat model and assumptions

This design primarily targets:
- Offline compromise of app storage (AsyncStorage copies, device backups, filesystem extraction)
- UI-level leakage where secrets pass through JS (logs/instrumentation/accidental persistence)
- Bridge-level tampering where JS tries to modify signing/reveal data without native/Rust verification

This design does **not** fully solve:
- Fully compromised device/OS (root/jailbreak/kernel-level compromise)
- Malicious accessibility overlays or hardware implants
- Side-channel attacks beyond typical mobile hardening
- Vulnerabilities in external dependencies, RPC providers, or chain infrastructure

---

## 13) What feedback we want (for Reddit review)

Most valuable review feedback typically falls into these categories:

1) **Opaque handle lifecycle safety**
- disposal correctness
- concurrency/threading pitfalls
- FFI boundary mistakes that could leak secrets

2) **Secret/reference stores**
- lifetime hazards
- crash/restart behavior
- risk of accidental persistence via logs, debug builds, managed strings

3) **Payload formats and versioning**
- forward/backward compatibility
- migration strategy and format agility
- domain separation / parameter binding / integrity checks

4) **Trusted UX**
- whether confirmation modals meaningfully reduce JS tampering risk
- secure input design pitfalls (IME, accessibility, screenshots)
- realistic constraints on screen capture and app switcher snapshots

---

## 14) Disclaimer

This documentation is provided “as is” without warranty of any kind. Use it for review and discussion, not as a substitute for professional security auditing.
