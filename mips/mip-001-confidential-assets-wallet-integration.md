---
mip: (this is determined by the MIP Manager, leave it empty when drafting)
title: Confidential Assets — Wallet Integration
author: ganymedio
discussions-to:
Status: Draft
last-call-end-date:
type: Standard (Interface)
created: 2026-05-17
updated:
requires:
---

# MIP-001 - Confidential Assets — Wallet Integration

**File naming convention:** When merged this file should be renamed to `mip-<NNN>-confidential-assets-wallet-integration.md` with the number assigned by the MIP Manager. Diagrams, if any, belong under `mips/diagrams/mip-<NNN>/`.

## Summary

The on-chain Confidential Assets protocol on Movement encrypts fungible-asset balances under per-account, per-token encryption keys (`ek[account, token]`) and verifies zero-knowledge proofs against them. The protocol is silent on how wallets custody the matching decryption keys (`dk`), how dApps request CA operations, or how multisig accounts — which have no private key — participate. Without an interface standard, every wallet and dApp re-implements derivation, storage, and the request surface, with byte-level divergence orphaning on-chain registrations and a real risk of `dk` leakage into web origins.

This MIP specifies the wallet ↔ dApp interface for Confidential Assets and the wallet-internal invariants every conforming wallet must hold. It defines:

- A `ca_*` dApp ↔ wallet method namespace (read and write) advertised through the wallet-standard `features` map.
- Canonical `dk` derivation policies for software, hardware, and keyless backings, each binding the token's FA metadata address into the derivation input.
- Per-`(account, token)` `dk` isolation, in-wallet proof construction, and an explicit-authorization model for rollover/normalization (no silent acceptance of incoming funds).
- A multisig participation model in which a single proposer constructs proofs against the multisig address (`mode: "buildOnly"`), and co-owners receive a shared per-vault `dk[Vault]` root through an ephemeral off-chain envelope share, from which every `dk[Vault, token]` is derived locally.
- Auditor inclusion rules (global, per-asset, per-transfer).

In scope: the wallet ↔ dApp interface, the wallet-side custody and proof-construction invariants, the canonical derivation policies for all three account backings, the per-vault shared-`dk` envelope protocol used to bootstrap multisig co-owners, and the minimum required changes to `@moveindustries/confidential-assets`.

### Out of scope

- Decryption-key rotation UI in conforming wallets. The protocol's `rotate_encryption_key` remains available through `@moveindustries/confidential-assets` for technical users; wallets expose no `ca_rotateEncryptionKey` method. Rotation in place addresses `dk`-only compromise. Mnemonic / device / pepper compromise is recovered by moving funds to a fresh account, not by in-place rotation.
- Defining a chain-level or per-asset auditor registry beyond consuming the existing `get_chain_auditor()` / `get_asset_auditor(token)` views.
- Threshold-ElGamal or MPC-based multi-owner custody. Considered in Alternative Solutions and rejected against the current on-chain verifier.
- Server-side coordinator services that hold `dk` on behalf of the user. Considered and rejected; violates the wallet-custody principle.
- New cryptographic primitives in the protocol. This MIP composes existing primitives (Twisted ElGamal, Bulletproofs, Sigma protocols, X25519 + HKDF + AES-GCM for the shared-`dk` envelope) under a new wallet/dApp interface. No new Move modules are introduced.

## High-level Overview

The proposed design is a clean split of responsibilities. Decryption keys live exclusively inside the wallet's privileged process (browser-extension background context or equivalent). Every ZK proof for register / transfer / withdraw / normalize is constructed in that process. The dApp sees only `ca_*` RPCs: it expresses intent ("transfer N of token T to address R"), and the wallet — after explicit user confirmation — fetches state, loads exactly one `dk[account, token]`, builds the proofs, and submits the transaction. The wallet adapter (`@moveindustries/wallet-adapter-react`) ships thin wrappers around these RPCs and bundles no CA SDK code into page JavaScript.

`dk` is scoped per `(account, token)` and derived canonically from each backing's root secret with the token's FA metadata address bound into the derivation input. Software backings derive at `m/44'/637'/{accountIndex}'/1'/{tokenIndex}'` where `tokenIndex = u32_le(SHA-256(tokenMetadataAddress)[0..4]) & 0x7FFFFFFF`. Hardware backings request a device signature over `decryptionKeyDerivationMessage ‖ ":" ‖ hex(tokenMetadataAddress)` and use `TwistedEd25519PrivateKey.fromSignature`. Keyless backings — which have no mnemonic and a rotating ephemeral key — derive over the per-identity **pepper** via `HKDF-SHA512(pepper, salt="movement-ca/v1", info="dk:" ‖ accountAddress ‖ tokenMetadataAddress, L=64)` reduced via a new `TwistedEd25519PrivateKey.fromUniformBytes` constructor. Multisig vaults use a different anchor: a single shared 32-byte `dk[Vault]` is bootstrapped to every co-owner once via the off-chain envelope share, and every `dk[Vault, token]` is derived locally as `HKDF-SHA512(dk[Vault], salt="movement-ca-vault/v1", info="dk:" ‖ multisigAddress ‖ tokenMetadataAddress, L=64)` reduced through the same `fromUniformBytes` path.

Multisig accounts hold funds but have no private key, so a single owner ("the dealer") generates a 32-byte per-vault root `dk[Vault]` once at vault setup. Every `dk[Vault, token]` is then derived locally from `dk[Vault]` and the token's FA metadata address; participants who hold `dk[Vault]` independently arrive at the same `dk[Vault, token]` for every registered and future asset without further coordination. Proofs are still constructed by a single proposer against the multisig address using `sender + mode: "buildOnly"` on the wallet RPC. To bootstrap co-owners, the dealer encrypts `dk[Vault]` per-recipient under each co-owner's X25519 key (birationally mapped from their on-chain Ed25519 owner key) and posts the envelopes to the multisig vault application's off-chain envelope store. Recipients fetch and decrypt under their owner key, persist `dk[Vault]` to a new per-vault local keystore entry, and the store deletes the envelopes once all recipients have read them. Rotation on a co-owner leak generates a fresh `dk[Vault]`, re-registers `ek` for every affected token via the standard SDK `rotate_encryption_key` flow, and re-shares — the wallet exposes no rotation UI.

## Impact

**Audiences and required actions:**

- **Wallet implementers (Motion Wallet, third-party Movement wallets).** Implement the `ca_*` namespace, the canonical derivation policies for every backing they support, the keystore-and-loader invariants, the multisig-entry data model, the explicit "Accept incoming funds" authorization for rollover, and the per-vault `dk[Vault]` share / fetch / decrypt / import flows. Wallets that do not support CA advertise the absence through wallet-standard feature detection; dApps degrade gracefully.
- **dApp developers.** Use `ca_*` RPCs via `@moveindustries/wallet-adapter-react` for every CA operation, including multisig. Stop bundling `@moveindustries/confidential-assets` or any derivative proof-construction code into page JavaScript. Treat `actual` and `pending` as distinct balances and prompt users to accept pending funds explicitly before retrying a spend on `INSUFFICIENT_BALANCE`. Conformance rules A1–A6 in [Specification and Implementation Details](#specification-and-implementation-details) are normative for any dApp claiming CA support.
- **Multisig vault application.** Operate the off-chain envelope store used to deliver `dk[Vault]` to co-owners: accept envelopes posted by the dealer, expose them to the addressed recipients only, and delete each envelope once all recipients have read it (or on explicit dealer revocation). The store is trusted for honest deletion, availability during the share window, and non-discriminatory delivery — it is not trusted for confidentiality (envelopes are end-to-end encrypted) or for fund safety (k-of-n Ed25519 still gates movement).
- **SDK maintainers (`@moveindustries/confidential-assets`, `@moveindustries/ts-sdk`).** Land the four required SDK changes in [SDK changes required by this design](#sdk-changes-required-by-this-design): remove auto-rollover from `withdrawWithTotalBalance` / `transferWithTotalBalance` (or delete the helpers), add a build-only proof-construction API, export canonical derivation helpers, and ship the per-vault `dk[Vault]` envelope helpers (encrypt / decrypt / `dk[Vault, token]` derivation). Add `TwistedEd25519PrivateKey.fromUniformBytes`.
- **Asset issuers and chain governance.** No required action for this MIP — but note that the wallet refuses to construct a confidential transfer if `get_chain_auditor()` returns `None`. Configuring a chain-level auditor is therefore a deployment prerequisite for end-to-end confidential transfers.
- **End users.** Visible behavior change for receive-only or pending-heavy accounts: incoming confidential transfers accumulate as a labeled "pending — accept incoming funds" line that the user explicitly authorizes before it becomes spendable. The wallet never bundles rollover into a spend.

**Dependencies.** None on prior MIPs. This MIP is the first MIP that touches the wallet ↔ dApp interface for CA. No new on-chain modules are required.

## Alternative Solutions

The most consequential design choice in this MIP is how multisig accounts participate in confidential assets — specifically, how the `dk` material needed to decrypt balances and construct proofs is custodied across co-owners. Four constructions were considered. The matrix below compares them; the chosen construction is the per-vault shared-`dk` model with off-chain ephemeral envelope distribution (the first row).

| Approach | Material per owner | Proof construction | Privacy under one-owner wallet compromise | Funds under one-owner wallet compromise | Viable against the current protocol |
|---|---|---|---|---|---|
| **Per-vault shared-`dk` with off-chain ephemeral envelope distribution (this MIP)** | Identical 32-byte `dk[Vault]` root per multisig vault (from which every `dk[Vault, token]` is locally derived), plus the owner's own Ed25519 key | One proposer constructs proofs using `dk[Vault, token]`; approvers contribute only Ed25519 signatures | Lost for every asset under the compromised vault's `dk[Vault]`; preserved for unrelated vaults and the owner's personal accounts | Safe — fund movement requires k-of-n Ed25519 signatures | Yes — requires no protocol changes. The envelope is delivered through the multisig vault application's off-chain store and deleted once all recipients have read it |
| Shared-`dk` with on-chain envelope delivery via a minimal new Move module (`dk_inbox`) | Per-asset `dk[multisig, token]` (one envelope per token over the vault's lifetime), plus the owner's own Ed25519 key | Same as the chosen approach | Lost for the asset whose `dk` is exposed; per-asset isolation preserves the rest | Safe | Yes — but envelopes are public chain calldata indefinitely. A later compromise of any recipient's owner key decrypts every historical envelope addressed to them. Rejected in favor of off-chain ephemeral delivery, which avoids the retroactive-decryption tail at the cost of trusting the store for honest deletion |
| Per-owner separate `dk` (re-encrypt to all owners) | The owner's own `dk`; transfers carry one ciphertext per owner | Proposer constructs proofs against multiple `ek` values; on-chain verifier checks all | Privacy lost only against the compromised owner; others retain it | Safe | No — current Move modules store one `ek` per `(account, token)` registration; would require protocol changes and break per-asset auditor accounting |
| Threshold ElGamal with threshold ZK (true MPC) | A share of `dk`; no single owner can decrypt | k owners run interactive MPC to jointly decrypt and construct a single proof | Preserved — attacker holds one share, below threshold | Safe | No — requires a threshold-ElGamal-aware Move verifier, threshold-friendly Bulletproofs/Sigma, and an MPC channel between wallets. Substantial protocol and wallet work |
| Trusted-coordinator service (server holds `dk`; owners authenticate to it) | The owner's own Ed25519 key; an auth token | Coordinator constructs proofs on owners' behalf | Lost on coordinator compromise — SPOF outside the wallet trust boundary | Safe — k-of-n approvals still required on chain | Possible to build, but rejected. Violates Principle 1: `dk` is wallet-custodied. Disclosure to a shared service outside the user's control is excluded by design |

The per-vault shared-`dk` model with off-chain ephemeral envelope distribution is the construction that is (a) viable against the current on-chain verifier with no protocol or framework changes, (b) consistent with the wallet-custody principle — envelopes are end-to-end encrypted and the store sees only ciphertext, and (c) shippable in a single coordinated release across wallet + SDK + multisig vault application. The trust assumption — that the off-chain envelope store honestly deletes envelopes once all recipients have read them, remains available during the share window, and does not discriminate between recipients — is documented in [Security Considerations](#security-considerations). Sharing once per vault (rather than once per asset) means every future `dk[Vault, token]` is derived locally and requires no further envelope traffic.

Two further alternatives were rejected at the operation level:

- **Auto-rollover on spend.** Today's SDK helpers `withdrawWithTotalBalance` / `transferWithTotalBalance` silently submit `rollover_pending_balance` to cover a shortfall. This conflates "spend" with "accept incoming funds," gives unsolicited senders a vector to extract gas, and violates the explicit-authorization principle. This MIP requires their removal or restriction (see [SDK changes required by this design](#sdk-changes-required-by-this-design)).
- **dApp-supplied derivation parameters.** Allowing a dApp to pass a derivation path prefix, signed-message prefix, or HKDF parameters would enable phishing to a wrong `ek` registration. The wallet accepts only the 32-byte FA metadata address from the dApp; every other derivation input is wallet-fixed.

## Specification and Implementation Details

### Guiding principles

1. **Decryption keys are wallet-custodied.** `dk` is stored in the wallet's encrypted keystore, used in-process for proof construction, and disclosed outside the wallet only through an explicit, user-initiated export flow. dApps, web origins, and the wallet adapter never receive `dk` bytes.
2. **Per-asset `dk` isolation.** The wallet derives, stores, and uses a distinct `dk` for every `(account, token)` pair. There is no per-account `dk`.
3. **Proof generation occurs inside the wallet.** Every ZK proof for registration, transfer, withdraw, and normalize is constructed in the wallet using `dk` for the asset being acted on. Key rotation is not wallet-supported.
4. **Rollover and normalization require explicit user authorization.** They incur gas and alter account state. The wallet surfaces pending balance with an explicit "Accept incoming funds" action and submits the transaction only after the user authorizes it.
5. **The application expresses intents; the user authorizes every transaction.** Every on-chain transaction is preceded by user confirmation in the wallet UI.

### Trust boundary

The wallet (privileged process) holds: Ed25519 signing key (software) or device pairing (hardware) or keyless pepper (keyless); the per-`(account, token)` `dk` set; the proof-construction code; balance-decryption code; transaction building/signing; rollover/normalize orchestration; and auditor key lookups. It exposes the `ca_*` interface to the application. The application holds nothing CA-secret and **must not** hold `dk`, build proofs, or call the CA SDK directly.

### Decryption key lifecycle

#### Scope

One `dk` per `(account, token)`. An account registered for n tokens holds n distinct `dk` values. The on-chain `ek` slot for each `(account, token)` registration is `dk[token].publicKey()`. Operations on asset X load `dk[X]`; `dk[Y]` for `Y ≠ X` remains sealed.

#### Canonical derivation policies

These layouts are part of the wallet ↔ chain compatibility contract. A different formula yields a different `dk` and orphans existing registrations. Wallet release notes must call out any change.

**Software backings (mnemonic).** Ed25519 signing key at `m/44'/637'/{accountIndex}'/0'/0'`. Per-asset `dk` at:

```
m/44'/637'/{accountIndex}'/1'/{tokenIndex}'

tokenIndex = u32_le(SHA-256(tokenMetadataAddress)[0..4]) & 0x7FFFFFFF
```

`dk[token] = TwistedEd25519PrivateKey.fromDerivationPath(path, mnemonic)`.

**Hardware backings (device signature):**

```
message[token] = decryptionKeyDerivationMessage ‖ ":" ‖ hex(tokenMetadataAddress)
dk[token]      = TwistedEd25519PrivateKey.fromSignature(device.sign(message[token]))
```

`decryptionKeyDerivationMessage` is the SDK-fixed constant `"Sign this message to derive decryption key from your private key"`. `hex(tokenMetadataAddress)` is the 32-byte address rendered as 64 lowercase hex characters with no `0x` prefix; the separator is a single ASCII colon. Each natively derived `dk` is recomputed from a fresh device signature on every wallet unlock; not persisted at rest.

**Keyless backings (pepper).** Keyless wallets have no mnemonic and a rotating ephemeral key; the stable per-identity secret is the **keyless pepper** already held for address derivation.

```
ikm  = pepper                                                          // 31 bytes for current pepper service;
                                                                       //   HKDF is robust to other widths
salt = utf8("movement-ca/v1")                                          // 14 bytes
info = utf8("dk:") || accountAddress || tokenMetadataAddress           // 3 + 32 + 32 = 67 bytes
                                                                       //   raw 32-byte addresses, not hex
okm  = HKDF-SHA512(ikm, salt, info, L = 64)                            // 64 bytes
dk[account, token] = TwistedEd25519PrivateKey.fromUniformBytes(okm)
```

`accountAddress` is the address whose on-chain `ek` slot this `dk` is derived for: the keyless account's own address when deriving an owner-account `dk`, or the multisig address when the keyless owner is acting as multisig proposer. Binding `accountAddress` into `info` is what lets a single keyless identity (one pepper) safely back the owner's own account plus any number of multisigs the owner proposes for, without `dk` collisions. The `v1` suffix in `salt` reserves room for a future `v2` layout.

**Federated keyless.** The federated-keyless pepper has identical semantics to the vanilla pepper (same lifecycle, no rotation, per-identity stability). The HKDF policy applies unchanged.

**dApp-supplied derivation input.** A dApp may supply the 32-byte FA metadata address through `ca_register`, `ca_transfer`, etc. It supplies no other derivation parameter. The wallet does not accept path prefixes, hardened-index counts, signed-message prefixes, HKDF salts/infos, or any other derivation input from a dApp.

#### Storage and export

| Operation | Behavior |
|---|---|
| At rest | One keystore entry per `(account, token)`, sealed under the wallet's key-encryption key. |
| In memory | Loaded only while an operation against the corresponding token is running; zeroed on wallet lock and idle timeout. Loading `dk[X]` does not decrypt `dk[Y]`. |
| Export | User-initiated UI action, scoped to one `(account, token)`. Gated by master-password re-prompt and typed asset-name confirmation. Returns a single 32-byte hex string. No bulk export. No dApp-callable export. |
| Import | User-initiated UI action, scoped to one `(account, token)`. Imported entries are labeled `imported`. |
| Backup | For software, mnemonic recovery reproduces every natively derived `dk[token]`. For hardware, re-pairing the same device reproduces them. For keyless, pepper recovery reproduces them. Imported `dk` entries are reproduced by **none** of these and must be retained out of band. |
| Display | The wallet may display `ek[token]`. `dk[token]` bytes appear only inside the export confirmation flow. |

#### Security invariants

- `dk[token]` is in memory only while an operation against that token runs; zeroed on lock alongside the rest of unlocked key material (signing keys, mnemonic, pepper).
- `dk[token]` bytes are never returned to any web origin and are never logged.
- `dk[token]` at rest is either (a) derivable on demand from root key material the wallet holds (mnemonic, device re-signing, or pepper), or (b) a user-imported standalone blob in the encrypted keystore. Form (b) is never written by a dApp-callable code path.
- Per-asset isolation is enforced in code: the loader takes `(accountAddress, tokenMetadataAddress)` and returns exactly one `dk`. No API returns "the account's `dk`" or "all `dk` values." Proof routines accept a single `dk` and a single token address; mismatches abort before any cryptographic work.
- The derivation policy is stable across releases. Any change is a hard fork of `dk[token]` / `ek[token]` for affected backings.
- The derivation message used with `fromSignature`, and the HKDF salt/info used with the keyless pepper, are hard-coded in the wallet. The dApp's only derivation influence is the 32-byte FA metadata address it passes through `ca_*`.

#### Motion Wallet keystore schema (informative)

Motion Wallet's concrete schema is one `chrome.storage.local` entry per `(walletEntryId, tokenAddrLowerHex)`: an AES-GCM ciphertext of the raw 32-byte `dk` with AAD `utf8("mv-dk-v1") || accountAddress || tokenMetaAddr`, sealed under the runtime second-layer key. The invariants in [Storage and export](#storage-and-export) and [Security invariants](#security-invariants) above are normative for any wallet; the concrete schema is normative only for Motion Wallet. Other implementations may choose a different schema as long as the invariants hold.

### Operation-by-operation design

The default `mode: "submit"` flow is described per operation. For multisig CA operations the dApp passes `sender = <multisigAddress>` and `mode: "buildOnly"`; the wallet stops at proof construction and returns BCS-encoded `EntryFunction` bytes instead of submitting (see [Multisig accounts](#multisig-accounts)).

#### Register

- dApp calls `ca_register({ token })`.
- The wallet derives `dk[token]` and computes `ek[token] = dk[token].publicKey()`.
- The wallet persists the keystore entry, or confirms an existing one for this `(account, token)` pair.
- The wallet constructs the Schnorr ZKPoK registration proof, then builds and signs the `register(sender, token, ek, commitment, response)` transaction.
- The wallet presents the transaction for user confirmation and submits after confirmation.
- Re-registering the same `(account, token)` pair must reuse the existing `dk[token]`; the wallet does not silently rotate it.

#### Deposit

- dApp calls `ca_deposit({ token, amount })`.
- The wallet routes to a single on-chain entrypoint based on registration and normalization state:
  - `register_and_deposit_and_rollover_pending_balance` — not registered.
  - `deposit_and_rollover_pending_balance` — registered, normalized.
  - `deposit_and_normalize_and_rollover_pending_balance` — registered, not normalized.
- The not-registered route derives `dk[token]` to produce `ek[token]` for the embedded register call.
- Every route results in **one** on-chain transaction with one user approval.

#### Withdraw

- dApp calls `ca_withdraw({ token, amount })`.
- The wallet decrypts the actual balance with `dk[token]`.
- If `actual < amount`, the wallet returns `INSUFFICIENT_BALANCE` and **does not** auto-rollover pending funds.
- Otherwise the wallet builds the sigma proof and range proof for the new balance.
- The wallet presents one transaction for user confirmation and submits after confirmation.

#### Confidential transfer

- dApp calls `ca_transfer({ token, recipient, amount, auditorAddresses?, senderAuditorHint? })`.
- The wallet decrypts the actual balance; returns `INSUFFICIENT_BALANCE` if `actual < amount`.
- The wallet fetches the recipient's `ek[token]` from chain.
- The wallet fetches the chain-level auditor `ek` via `get_chain_auditor()` (mandatory inclusion) and the per-asset auditor via `get_asset_auditor(token)` (when configured).
- The wallet combines those with any per-transfer auditors from the request and builds the `ConfidentialTransfer` payload (sigma + two range proofs).
- The wallet presents one transaction for user confirmation and submits after confirmation.
- `senderAuditorHint`, when supplied, is bound into the sigma Fiat–Shamir transcript via the same `bcs::to_bytes(sender_auditor_hint)` encoding the on-chain verifier uses. The wallet enforces the on-chain length cap from `max_sender_auditor_hint_bytes()`.

#### Rollover and normalization

- dApp calls `ca_rolloverPending({ token })` to express the user's intent to accept pending funds.
- The wallet computes whether `normalize` is required and chains it within a single user-confirmation step. Normalization is silent here because it is a protocol detail of "accept incoming funds."
- At most two on-chain transactions are submitted (`normalize` then `rollover`); the final `rollover` hash is returned.
- The wallet **never** initiates rollover or normalization on a timer, on balance fetch, on inbound transfer, or in response to any dApp signal.
- When `pending > 0` the wallet displays a distinct "Accept incoming funds" action.
- `ca_withdraw` and `ca_transfer` never bundle rollover.

#### Key rotation

- Not wallet-supported.
- Motion Wallet exposes no UI for rotation and no `ca_rotateEncryptionKey` method.
- A technical user requiring same-account rotation uses `@moveindustries/confidential-assets` (`ConfidentialAsset` / `ConfidentialAssetTransactionBuilder.rotateEncryptionKey`) directly.
- Rotation in place addresses only `dk`-only compromise; mnemonic / device / pepper compromise is recovered by moving funds to a fresh account.

### Wallet UX decisions

- **Balance visibility.** Confidential balances are shown by default as a separate line item beneath the regular asset, not hidden behind a toggle. "Confidential" refers to on-chain privacy, not to visual concealment from the account holder.
- **Pending vs spendable.** `actual` (spendable) and `pending` (awaiting acceptance) are displayed as separate fields, never summed into a single "total balance" number.
- **Spam tokens.** Inbound assets accumulate in pending and remain visible until the user activates "Accept incoming funds." No on-chain transaction is incurred for unsolicited tokens unless the user opts in. Unknown tokens get a warning badge; known assets may default to single-tap confirmation.

### Hardware wallets

Motion Wallet is expected to be able to back an account with a hardware device in the future. The mnemonic stays on the device, so `dk[token]` is derived via `fromSignature` rather than `fromDerivationPath`. Each natively derived `dk[token]` is recomputed from a fresh device signature on every unlock and is not persisted at rest. Imported `dk` entries are persisted in the encrypted keystore.

**Security properties.** The Ed25519 signing key never leaves the device; fund movement requires a physical button press. During a CA operation the loaded `dk[token]` resides in wallet memory and is exposed to a wallet-process compromise during that window — disclosing the balance for that token and enabling valid CA proofs against `ek[token]`. Those proofs still need a device-signed transaction to execute, so **funds remain safe**; the loss is confined to privacy of the tokens whose `dk` was loaded during the compromise window. The wallet UI must not represent confidential balances as device-protected.

**Device requirement.** The chain application must expose deterministic message signing over arbitrary fixed byte strings. CA support is unavailable against any hardware backing that does not provide this.

### Keyless accounts

A keyless account has no mnemonic and a rotating ephemeral key; the wallet derives `dk[token]` from the **keyless pepper** via the HKDF policy in [Canonical derivation policies](#canonical-derivation-policies). The pepper is the only client-side secret that is both stable across rotations and unique per identity, so it is the correct anchor.

**Why the pepper, not the ephemeral key.** Using the ephemeral key would produce a different `dk` after every rotation, orphaning the registered `ek` and rendering the confidential balance unrecoverable.

**Security properties.** Fund movement requires a valid OIDC proof and an ephemeral-key signature. As with hardware backings, a wallet-process compromise during a CA operation discloses the balance for the active token; funds remain safe. The pepper sits in process memory while the wallet is unlocked and a CA operation runs — a pepper compromise is equivalent to a compromise of every `dk[token]` for that account.

**Recovery.** Pepper recovery (the same path that recovers the keyless address) reproduces every natively derived `dk[token]`. Pepper loss is equivalent to mnemonic loss for software backings. Loss of OIDC provider access blocks pepper recovery — the on-chain keyless account faces the same fate for fund movement, so CA recovery inherits the same long-tail risk as keyless itself rather than introducing a new one. Wallet UX must make this risk obvious before users store meaningful confidential balances in a keyless-only account.

**Cross-device determinism** depends on the pepper service returning byte-identical pepper for the same `(iss, aud, uid_key, uid_val)` tuple, regardless of which device is asking — the analog of software backings' "same mnemonic on every device."

**OIDC identity tuple not bound into HKDF info.** The pepper is already a deterministic function of `(iss, aud, uid_key, uid_val)`; redoubling the binding in `info` would be redundant and would close off the future option of non-OIDC pepper recovery (e.g. social recovery).

### Multisig accounts

A multisig account is a resource account: it holds funds but has no private key. Proofs must bind to the multisig address — the SDK's Fiat–Shamir transcript includes `senderAddress` — and proofs built against any other address abort on chain.

**Wallet entry model.** Motion Wallet represents a multisig as a first-class wallet entry:

```ts
type WalletEntry =
  | { kind: 'mnemonic';    id: string; /* … */ }
  | { kind: 'private-key'; id: string; /* … */ }
  | { kind: 'keyless';     id: string; /* … */ }
  | {
      kind: 'multisig';
      id: string;
      address: string;
      threshold: number;
      owners: string[];
      ownedByWalletIds: string[];  // local Ed25519 wallets that co-own;
                                   // can be empty (view-only)
    };
```

A multisig entry is a **view-only reference**, not a switchable signing account: there is no private key, and the dApp-visible connected account always remains the user's Ed25519 wallet. Selecting a multisig entry opens its dk-management surface (derive / import / export per token, view balances); it does **not** change `getAccount()` for dApps, because every fund-moving call — propose, approve, execute — is signed by the owner's Ed25519 key against `0x1::multisig_account::*` with the multisig address as a function argument. The wallet identifies the target multisig for `ca_*` `buildOnly` calls by matching the dApp-supplied `sender` parameter against `address` on a multisig entry; the entry does not need to be "active."

**Data ownership:**

| Held by | Material | Used for |
|---|---|---|
| Each owner (private) | Owner's mnemonic / device / pepper | Producing the owner's Ed25519 signatures on multisig proposals |
| Each owner (private) | Owner's Ed25519 signing key | Approving / rejecting multisig proposals on chain |
| Every owner (shared, identical bytes) | `dk[Vault]` — 32-byte per-vault root | Locally deriving `dk[Vault, token]` for every registered and future asset; bootstrapped once via the off-chain envelope share |
| Every owner (derived locally from `dk[Vault]`) | `dk[Vault, token]` per registered token | Decrypting multisig confidential balance for that token; building proofs against the multisig address |
| Off-chain (ephemeral, multisig vault application) | Envelopes encrypting `dk[Vault]` to each co-owner's X25519 key | Bootstrapping co-owners; deleted once all recipients have read |
| On chain (public) | Multisig address, owners, threshold | k-of-n authorization |
| On chain (public) | Multisig's per-token `ek[token]` | Lets senders encrypt confidential transfers to this multisig |
| On chain (public, encrypted) | Multisig's confidential balances under `ek` | Source of truth for confidential balances |

**`dk[Vault]` lifecycle:**

- **Generation.** At vault setup, one designated owner (the "dealer") generates `dk[Vault]` as 32 uniformly random bytes from the wallet's CSPRNG. `dk[Vault]` has no on-chain footprint and is independent of any owner's root key material — it cannot be reconstructed from any owner's mnemonic / device / pepper alone, so loss of `dk[Vault]` across all owners is unrecoverable.
- **Local storage.** Each participant's wallet holds one new keystore entry per vault, sealed under the wallet's key-encryption key: `dk[Vault]` keyed by `multisigAddress`. The existing per-`(account, token)` keystore continues to hold `dk[Vault, token]` entries that the wallet either caches on first derivation or recomputes on demand.
- **Per-asset derivation (every backing, including multisig).** Once a participant holds `dk[Vault]`, every asset's `dk[Vault, token]` is derived locally — no further envelope traffic is required for new assets the vault registers:

```
ikm  = dk[Vault]                                                       // 32 bytes
salt = utf8("movement-ca-vault/v1")                                    // 22 bytes
info = utf8("dk:") || multisigAddress || tokenMetadataAddress          // 3 + 32 + 32 = 67 bytes
                                                                       //   raw 32-byte addresses, not hex
okm  = HKDF-SHA512(ikm, salt, info, L = 64)                            // 64 bytes
dk[Vault, token] = TwistedEd25519PrivateKey.fromUniformBytes(okm)
```

Binding `multisigAddress` into `info` makes the derivation domain-separated even in the unlikely event two vaults end up with the same random `dk[Vault]`. The `v1` suffix in `salt` reserves room for a future `v2` layout.

**Wallet API requirements.** Every `ca_*` write method accepts:

- `sender?: string` — the address bound into the Fiat–Shamir transcript. Default: wallet's own account. For multisig: the multisig address.
- `mode?: "submit" | "buildOnly"` — `"buildOnly"` returns BCS-encoded `EntryFunction` bytes for `MultiSigTransactionPayload`. Default: `"submit"`.

A request with `sender ≠ wallet account` MUST set `mode: "buildOnly"`. The wallet rejects `{ sender: <other>, mode: "submit" }`.

**Transfer flow** (off-chain proof build + on-chain k-of-n approval):

```
App  ──ca_transfer sender=multisigAddr, mode=buildOnly──▶  Owner 1 wallet
Owner 1 wallet ── read ek, balance, recipient ek, auditors ──▶ Chain
Owner 1 wallet ── decrypt with dk[Vault, token]; build proofs bound to multisigAddr ──▶ self
Owner 1 wallet ──── BCS EntryFunction bytes ────▶ App
App  ──── multisig_account::create_transaction (Owner 1 Ed25519 sig) ────▶ Chain
Owner 2..k wallets ──── approve_transaction (Owner i Ed25519 sigs) ────▶ Chain
Chain executes after k approvals, verifies proofs.
```

Only the proposer needs `dk[Vault, token]` for the in-flight proposal. Approvers verify on-chain semantics through their wallet UI (recipient, amount ciphertext, auditor inclusion, proposal hash) and do not re-construct proofs. Every owner who holds `dk[Vault]` can locally derive `dk[Vault, token]` for any asset, so any qualified owner may serve as a future proposer and may locally decrypt balances for audit.

**`dk[Vault]` sharing among co-owners.** Other owners cannot reproduce the dealer's `dk[Vault]` from their own root material — it is uniformly random and held only by the dealer until the share completes. Bootstrapping co-owners therefore requires an authenticated one-shot delivery of `dk[Vault]` from the dealer to every other owner. Per-asset `dk[Vault, token]` is then derived locally by every holder of `dk[Vault]` and requires no further sharing for assets the vault later registers.

**Off-chain envelope delivery.** Sharing happens through the multisig vault application's off-chain envelope store. The store is a per-vault hosted resource (operated by the multisig vault application) that accepts envelopes posted by the dealer, exposes them to the addressed recipients only, and deletes each envelope once all recipients have read it or on explicit dealer revocation. The store sees only ciphertext — `dk[Vault]` is end-to-end encrypted to each recipient and confidentiality does not depend on the store. Nothing about the envelope share is published on chain.

**Envelope format.** A single envelope carries one ephemeral X25519 public key plus one per-recipient AES-GCM ciphertext for every co-owner:

```
envelope_v1 :=
    "mv-dk-vault-v1"          // 14-byte ASCII version tag, also included in AAD
  ‖ multisigAddress           // 32 raw bytes
  ‖ ephemeralX25519Pub        // 32 bytes — dealer's per-share ephemeral X25519 public key
  ‖ recipientCount            // u16 little-endian
  ‖ for each recipient i:
        recipientOwnerAddress // 32 raw bytes
      ‖ nonce_i               // 12 bytes — fresh random AES-GCM nonce per recipient
      ‖ ciphertextWithTag_i   // 48 bytes = 32-byte dk[Vault] + 16-byte GCM tag

AAD_i := utf8("mv-dk-vault-v1")
       ‖ multisigAddress         (32 raw bytes)
       ‖ dealerOwnerAddress      (32 raw bytes)
       ‖ recipientOwnerAddress   (32 raw bytes)
       ‖ ephemeralX25519Pub      (32 raw bytes)
```

Recipient X25519 pubkey is the standard birational map of the on-chain Ed25519 owner pubkey. `sharedSecret_i = X25519(ephemeralPriv, recipientX25519Pub_i)`. `aesKey_i = HKDF-SHA256(sharedSecret_i, salt = utf8("mv-dk-vault-v1"), info = utf8("mv-dk-vault-v1") ‖ multisig ‖ dealer ‖ recipient_i ‖ ephemeralX25519Pub, L = 32)`. AES-GCM-256 seals the 32 bytes of `dk[Vault]` under `aesKey_i` with `AAD_i` and the per-recipient random nonce. Per-recipient `aesKey_i` are forgotten by the dealer immediately after sealing.

**Initial share flow.** The dealer's wallet generates `dk[Vault]`, fetches the multisig's owner set from chain and reads each owner's on-chain Ed25519 pubkey, builds one envelope with one ciphertext slot per co-owner, and posts it to the multisig vault application's envelope store under the vault's identifier. The application surfaces a "Pending dk share" notification on each recipient's view of the vault. Each recipient's wallet fetches the envelope, parses out its own `(nonce_i, ciphertextWithTag_i)`, decrypts with its X25519 key (derived from the owner's Ed25519 signing key), persists `dk[Vault]` to the per-vault keystore entry, and acknowledges the read to the store. When the store sees acknowledgements from every recipient, it deletes the envelope. The dealer may also explicitly revoke and re-share if the share window stalls.

An owner who has never transacted (no on-chain Ed25519 pubkey) is omitted from the initial share and re-shared later via a single-recipient envelope as soon as their pubkey is on chain.

**Owner additions and removals:**

- **Owner added.** The dk-management surface diffs the on-chain owner set against the local `dk[Vault]` presence. When the wallet detects an owner without `dk[Vault]`, it surfaces a "Share vault dk with new owner 0xefgh…" action, gated by explicit confirmation that surfaces the new owner's address. The share posts a length-1 envelope to the off-chain store.
- **Owner removed.** A removed owner retains `dk[Vault]` locally — neither the chain nor the multisig vault application can reach into their wallet — and retains decryption capability for past and future ciphertexts under every registered `ek[Vault, token]`. Funds remain safe (k-of-n approvals still required). Remaining owners generate a fresh `dk[Vault]`, rotate `ek` per affected token via the SDK's `rotate_encryption_key`, and re-share the new `dk[Vault]` to the remaining owners. The wallet detects the owner-set change on chain and surfaces a banner listing the rotation/re-share work pending.

The wallet must warn before proposing removal of an owner if `dk[Vault]` is not currently held by at least one remaining owner — rotation requires the current `dk[Vault]` to construct its proofs.

**Discovery.** The wallet does **not** poll. Each multisig entry's dk-management surface exposes an explicit **"Check for shared dks"** action that queries the multisig vault application's envelope store for envelopes addressed to the connected owner under this vault, decrypts each, and presents an import confirmation per result. A locally missing `dk[Vault]` for a vault the user has been added to surfaces as a banner.

**Import dialog.** Rendered in wallet-popup chrome (same trust surface as the transaction signing dialog), never as web content. States dealer owner address, multisig address, and a typed-confirmation gate matching the vault label. Accepts two formats: `mv-dk-vault-v1:` envelopes from the off-chain store (primary), and `mv-dk-vault-raw-v1:` raw hex of `dk[Vault]` (manual import, e.g. for vault recovery from a backed-up secret stored out of band).

### Auditor support

**Three kinds of auditors compose on a single transfer:**

1. **Global (chain-level) auditor.** Configured at chain level via `set_chain_auditor`; applies to every confidential transfer with no exceptions. Read via `get_chain_auditor()` (and `get_chain_auditor_epoch()` for staleness checks). The wallet **must** include it in every confidential transfer and **must refuse** to construct a transfer when none is configured.
2. **Per-asset auditor.** Optional; configured per fungible asset by the asset issuer via `set_asset_auditor`. Read via `get_asset_auditor(token)` (and `get_asset_auditor_epoch(token)`). The wallet must include it in transfers of the affected asset when configured.
3. **Per-transfer (voluntary) auditors.** The sender may include additional auditor `ek` values at transfer time. Not stored on chain.

**Wallet responsibilities.** Read both auditors from chain on every transfer; build encrypted copies of the transfer amount for each included key (sigma-bound via Fiat–Shamir, as the SDK handles); surface in the user's review step the full set of auditors that will receive a copy; expose the global and per-asset auditors for the user to view independently of any pending transfer.

**Auditor epoch.** When `auditor_epoch` is exposed on chain, the wallet reads it alongside the key, treats a mismatch with any cached value as stale, and refreshes before constructing a transfer.

### Wallet ↔ application interface

#### Method namespace

The `ca_*` methods are the dApp ↔ wallet RPC surface. They are not exports of the TypeScript SDK; wallet support for each is implementation-defined and advertised via wallet-standard features (below).

**Read methods:**

| Method | Request | Response | Notes |
|---|---|---|---|
| `ca_getBalances` | `{ tokens: string[] }` | `{ balances: { token, registered, available, pending }[] }` | Wallet decrypts; dApp sees plaintext numbers only |
| `ca_isRegistered` | `{ token }` | `{ registered: boolean }` | No `dk` needed |
| `ca_getEncryptionKey` | `{ token }` | `{ encryptionKey: string }` | Public; safe to return |
| `ca_getGlobalAuditor` | `{}` | `{ auditorEncryptionKey?: string, epoch: number }` | Chain-level. `auditorEncryptionKey` omitted if unconfigured (wallet then refuses to construct transfers). |
| `ca_getAuditor` | `{ token }` | `{ auditorEncryptionKey?: string, epoch: number }` | Per-asset. Omitted if unconfigured for that token. |

**Write methods:**

| Method | Request | Response | Notes |
|---|---|---|---|
| `ca_register` | `{ token, sender?, mode? }` | `{ hash }` or `{ entryFunctionBytes }` | Derive `dk`, build proof, confirm, submit (or return BCS bytes if `buildOnly`). |
| `ca_deposit` | `{ token, amount, sender?, mode? }` | `{ hash }` or `{ entryFunctionBytes }` | Routes to one of three on-chain entrypoints per state; always one transaction. |
| `ca_withdraw` | `{ token, amount, sender?, mode? }` | `{ hash }` or `{ entryFunctionBytes }` | Actual balance only; `INSUFFICIENT_BALANCE` if `amount > actual` (pending not counted). |
| `ca_transfer` | `{ token, recipient, amount, auditorAddresses?, senderAuditorHint?, sender?, mode? }` | `{ hash }` or `{ entryFunctionBytes }` | Same `INSUFFICIENT_BALANCE` semantics as `ca_withdraw`. |
| `ca_rolloverPending` | `{ token, sender?, mode? }` | `{ hash }` or `{ entryFunctionBytes }` | Accept incoming funds. Chains `normalize` if required, silently within one approval. Returns the final `rollover` hash. |

`sender` defaults to the wallet's own account. Non-default `sender` requires `mode: "buildOnly"`. `mode` defaults to `"submit"`.

`senderAuditorHint`, when supplied, is BCS `vector<u8>`-encoded (ULEB128 length prefix + bytes) into the sigma Fiat–Shamir transcript — exactly as the on-chain verifier does (`bcs::to_bytes(sender_auditor_hint)` in `confidential_proof.move`). The wallet enforces `max_sender_auditor_hint_bytes()` before proof construction.

**Errors.** Failed calls return:

```ts
type CaError = {
  code: CaErrorCode;
  message: string;             // user-facing, no internals
  details?: CaErrorDetails;    // only the fields below
};

type CaErrorCode =
  | 'USER_REJECTED' | 'WALLET_LOCKED' | 'NOT_CONNECTED'
  | 'UNSUPPORTED_METHOD' | 'UNSUPPORTED_MODE' | 'CA_FEATURE_UNAVAILABLE'
  | 'INVALID_REQUEST' | 'TOKEN_NOT_REGISTERED' | 'TOKEN_FROZEN' | 'TOKEN_DISABLED'
  | 'INSUFFICIENT_BALANCE' | 'PENDING_COUNTER_LIMIT'
  | 'NETWORK_ERROR' | 'CHAIN_REJECTED' | 'PROOF_FAILED' | 'INTERNAL_ERROR';

type CaErrorDetails = {
  abortCode?: number;
  moduleAbort?: string;          // e.g. "ENORMALIZATION_REQUIRED"
  txHash?: string;
  requiredCapability?: string;   // e.g. "ca_transfer", "buildOnly"
};
```

The wallet **does not** surface: internal storage paths, vault format or KDF parameters, balance ciphertexts or decrypted values outside `ca_getBalances`, proof intermediates, hardware-device responses beyond "device error" / "user rejected on device", stack traces, or any field that varies with private state. `INTERNAL_ERROR` is intentionally opaque (a correlation id may be surfaced for support). `CaErrorCode` is stable across releases; additions follow the same versioning as adding methods.

**Concurrency.** `ca_*` writes require explicit user authorization. The user is one person at one confirmation surface — no UX state admits two writes in parallel; no per-account mutex is required. Reads run concurrently and may return slightly stale data. For the window between "user authorized A" and "A confirmed on chain," a second `ca_*` call builds against fresh on-chain state; if A subsequently lands first, the chain rejects the second with `CHAIN_REJECTED` and the dApp retries. Self-correcting.

**Wallet adapter integration.** `@moveindustries/wallet-adapter-react` exposes thin wrapper functions:

```ts
const {
  caTransfer,
  caGetBalances,
  caGetGlobalAuditor,
  caGetAuditor,
  caSupported,
} = useConfidentialAssets();
```

These are RPC calls to the wallet, not invocations of the CA SDK in the browser. The adapter MUST NOT offer a generic "sign arbitrary bytes for confidential assets" hook — when `dk` is derived via `fromSignature`, the signed payload is wallet-fixed; allowing dApp-supplied payloads enables phishing to a wrong-`ek` registration. The same constraint applies to keyless HKDF parameters.

**Wallet-standard feature advertisement.** Confidential-asset support is published under **two keys pointing at the same feature object**, matching the dual-publish convention for every other feature in the Movement adapter:

```ts
features: {
  // ...
  'aptos:confidentialAssets':    confidentialAssetsFeature,
  'movement:confidentialAssets': confidentialAssetsFeature,
  // ...
}
```

No version suffix on the feature key. A future incompatible change will adopt whatever versioning convention the wallet-standard has adopted by then (for example a `:v2` suffix), with co-publication during a deprecation window.

#### SDK changes required by this design

The `@moveindustries/confidential-assets` package needs four changes for this MIP to be implementable in conforming wallets:

1. **`withdrawWithTotalBalance` / `transferWithTotalBalance` must not auto-rollover.** Either delete the helpers (recommended) or rename them and remove the auto-rollover behavior, so they throw `INSUFFICIENT_BALANCE` whenever `actual < amount`, regardless of pending. Restores the invariant that no SDK code path silently accepts incoming funds.
2. **Build-only API for proof construction.** Add `buildRegister` / `buildDeposit` / `buildWithdraw` / `buildConfidentialTransfer` / `buildRolloverPending` / `buildNormalize` (or a sibling `ConfidentialAssetBuilder` class). Each takes an explicit `sender: AccountAddressInput` and a `decryptionKey`, no signer, no fee payer, returns `Uint8Array` of BCS-encoded `EntryFunction` bytes. Required for multisig.
3. **Canonical derivation helpers.** Export `tokenIndexFromMetadataAddress`, `softwareDecryptionKeyDerivationPath(accountIndex, tokenMetaAddr)`, `hardwareDecryptionKeyDerivationMessage(tokenMetaAddr)`, `keylessDecryptionKey(pepper, accountAddress, tokenMetaAddr)`, and `vaultDecryptionKey(dkVault, multisigAddress, tokenMetaAddr)` (the HKDF-SHA512 step from `dk[Vault]` to `dk[Vault, token]`). Add a new `TwistedEd25519PrivateKey.fromUniformBytes(bytes: Uint8Array)` constructor that accepts ≥ 32 bytes of uniform input and reduces modulo the Ed25519 group order ℓ (mirrors the reduction inside `fromSignature`). Test vectors in the SDK pin the byte layouts so a regression is caught upstream of any on-chain registration.
4. **Per-vault `dk[Vault]` envelope helpers.** Export `sealVaultDk({ dkVault, multisigAddress, dealerOwnerAddress, recipients: { ownerAddress, ed25519PublicKey }[] })` returning the `mv-dk-vault-v1` envelope bytes, and `openVaultDk({ envelope, multisigAddress, recipientOwnerAddress, recipientEd25519PrivateKey })` returning the 32-byte `dk[Vault]`. Both helpers handle the X25519 birational map, HKDF-SHA256 key schedule, and AES-GCM-256 sealing/opening with the AAD layout in [Multisig accounts](#multisig-accounts). Test vectors pin every byte of the envelope layout.

#### Token addressing

All `ca_*` methods that take a `token` parameter use the **fungible-asset metadata object address** (32 bytes). Legacy coin type strings (the `0x1::module::CoinType` form) MUST NOT be used.

### Application conformance rules

| ID | Rule |
|---|---|
| A1 | dApps MUST NOT hold the user's Ed25519 signing private key. `ek` registration is wallet-only via `ca_register`. |
| A2 | dApps MUST NOT obtain, derive, or hold `TwistedEd25519PrivateKey` in the dApp process. They MUST NOT run the CA SDK for proof construction or balance decryption in page JS. They MUST use `ca_*` for all CA operations, including multisig (via `sender` + `mode: "buildOnly"`). |
| A3 | dApps MUST NOT persist, log, or forward CA decryption-key material. They MUST NOT ask the wallet to export `TwistedEd25519PrivateKey` to the page. |
| A4 | dApps MUST NOT derive `TwistedEd25519PrivateKey` in the page (`fromDerivationPath`, `fromSignature`, HKDF, or otherwise). CA key derivation is wallet-internal. |
| A5 | dApps MUST pass FA metadata addresses for `token`. |
| A6 | Deposit and withdraw amounts are public on chain; dApps MUST NOT imply that confidential transfer amounts are visible. |

## Reference Implementation

- **Wallet reference implementation:** `MoveIndustries/motion-wallet`, branch `feat/confidential-assets`. Notable surfaces:
  - `src/services/wallet/account.ts` — `WalletEntry`, multisig entry, `loadDk` loader contract, proposer-side derivation helpers.
  - `src/services/wallet/confidential-asset.ts` — `ca_*` handlers.
  - `src/services/wallet/keyless-session.ts`, `keyless-auth.ts`, `keyless-signer.ts` — keyless pepper handling via `@eigerco/movement-keyless`.
- **SDK reference:** `MoveIndustries/ts-sdk/confidential-assets`. The build-only API, the auto-rollover removal, the canonical derivation helpers (including `fromUniformBytes` and `vaultDecryptionKey`), and the per-vault envelope `sealVaultDk` / `openVaultDk` helpers are tracked as part of this MIP's required SDK changes.
- **Multisig vault application reference:** the multisig vault application gains a per-vault envelope store endpoint that accepts dealer-posted envelopes, exposes them to addressed recipients, tracks per-recipient acknowledgements, and deletes envelopes once all recipients have read or on explicit dealer revocation.

**Feature flag / enablement.** No node-level feature flag is required. Adoption is gated by:

- The multisig vault application release that exposes the envelope store endpoint (required for multisig CA bootstrapping).
- Chain governance configuring `set_chain_auditor` (required for conforming wallets to construct any confidential transfer).
- Wallets advertising `movement:confidentialAssets` / `aptos:confidentialAssets` in their wallet-standard `features` map.

dApps detect support through `caSupported` on the adapter; the absence of the feature key is a hard "not supported" signal.

## Testing

The testing plan is split across three layers, matching where the invariants live.

- **SDK (`@moveindustries/confidential-assets`).**
  - Fixed test vectors for every canonical derivation helper: software-path strings for representative `(accountIndex, tokenMetaAddr)` tuples, hardware signed-message bytes, HKDF okm bytes for representative `(pepper, accountAddress, tokenMetaAddr)` tuples (including federated-keyless cases), and `vaultDecryptionKey` okm bytes for representative `(dkVault, multisigAddress, tokenMetaAddr)` tuples. A regression that changes any byte fails CI before reaching on-chain registration.
  - Tests for `TwistedEd25519PrivateKey.fromUniformBytes` against `fromSignature`'s reduction over identical input bytes.
  - End-to-end proof construction via the new build-only API, with the resulting BCS bytes round-tripped through `MultiSigTransactionPayload` and verified on a local testnet.
  - Negative test: `withdrawWithTotalBalance` / `transferWithTotalBalance` either absent (option A) or throwing on `actual < amount` regardless of pending (option B).
  - Envelope round-trip: `sealVaultDk` followed by `openVaultDk` recovers the original 32-byte `dk[Vault]` for every addressed recipient; AAD mismatch (wrong `multisigAddress`, `dealerOwnerAddress`, or `recipientOwnerAddress`) causes decryption to fail; per-recipient `nonce_i` uniqueness within an envelope.
- **Wallet (Motion Wallet reference implementation).**
  - Unit tests on the keystore loader: `loadDk(accountAddress, tokenMetaAddr)` returns exactly one `dk`; mismatch between loader inputs and proof inputs aborts before any cryptographic work; AAD-bound ciphertext fails decryption when moved across `(account, token)` stores.
  - Lifecycle tests: lock / idle-timeout zeroes every cached `dk`, the `dk[Vault]` cache, the signing-key cache, the mnemonic (where present), and the pepper (where present).
  - `ca_*` end-to-end tests against a local node for register / deposit / withdraw / transfer / rollover, including the three deposit routes (not registered, registered+normalized, registered+not-normalized) collapsing to a single transaction and a single approval each.
  - Multisig: end-to-end `dk[Vault]` envelope share with N recipients via a local stub of the off-chain envelope store; recipient fetch + decrypt + import; deletion after all recipient acknowledgements; envelope decryption rejecting AAD mismatch; rotation triggered by owner removal (fresh `dk[Vault]` + `rotate_encryption_key` per token + re-share).
  - Local derivation of `dk[Vault, token]` from `dk[Vault]` produces byte-identical results across participants for the same `(multisigAddress, tokenMetaAddr)`.
  - Adapter integration: `caSupported` is `false` against a non-CA wallet; `useConfidentialAssets` falls back cleanly.
- **Multisig vault application envelope store.**
  - Acknowledgement accounting: an envelope is deleted only after every addressed recipient acknowledges; partial acknowledgement leaves the envelope retrievable by remaining recipients.
  - Access control: only the addressed recipient (authenticated as the owner of that address) can fetch their slot of the envelope.
  - Dealer revocation removes the envelope immediately and surfaces a re-share prompt in dealer wallets.

**When results are expected.** SDK and wallet unit/integration tests are gated by CI on each PR. Devnet rehearsals of `register → deposit → transfer → rollover` cycles run for single-owner and 2-of-3 multisig accounts before testnet promotion.

**Load testing** is not in scope: every `ca_*` write is preceded by a user approval, so contention is bounded by the user, not by request volume.

## Risks and Drawbacks

**Backward compatibility:**

- The `ca_*` namespace is additive. Wallets without CA support are unaffected; dApps without CA support are unaffected.
- The required removal (or restriction) of `withdrawWithTotalBalance` / `transferWithTotalBalance` is a breaking change to direct SDK callers. Callers should be migrated to `getBalance` + `withdraw` / `transfer`. The breaking change is intentional and is the only way to restore the explicit-authorization invariant; a soft-deprecation period in the SDK is feasible.
- The canonical derivation policies (software path, hardware message, keyless HKDF) are pinned forever from the first conforming release. A future policy change (for example a different HKDF salt) would orphan every existing registration; rotation in place is the only migration, and the SDK's `rotate_encryption_key` is the tool. The keyless HKDF salt `movement-ca/v1` carries an embedded version tag that reserves room for that future migration.

**Risks:**

| Risk | Mitigation |
|---|---|
| `dk[token]` lost (mnemonic / device / pepper loss). | Confidential balance for that token is unrecoverable for the affected account. Other tokens are unaffected. Recovery is per-backing (mnemonic restore, device re-pair, pepper recovery). Imported `dk` entries and multi-owner `dk[Vault]` roots are not reproduced by any backup path and must be retained out of band — wallets must label this clearly. |
| `dk[token]` compromised (malware, leaked export). | Privacy lost for that token only; per-asset isolation contains the scope of the compromise. Funds remain safe — `dk` alone cannot sign. Recovery is `rotate_encryption_key` for the affected `(account, token)` via the SDK, or moving funds to a fresh account when the broader root material is suspect. |
| `dk[Vault]` compromised on any participant. | Privacy lost for every asset under that vault — `dk[Vault]` derives every `dk[Vault, token]` locally. Funds remain safe (k-of-n approvals still required). Recovery is generating a fresh `dk[Vault]`, rotating `ek` per affected token via `rotate_encryption_key`, and re-sharing through the envelope store. |
| Cross-wallet divergence in derivation byte layouts (different `tokenIndex` formula, different signed-message prefix, different HKDF info). | Canonical derivation helpers exported from `@moveindustries/confidential-assets` with fixed test vectors. Wallets call helpers rather than re-implementing byte assembly; CI catches drift in the SDK before any registration. |
| Auto-rollover side effect in SDK (current `withdrawWithTotalBalance` / `transferWithTotalBalance`). | Required removal or restriction as part of this MIP. |
| Off-chain envelope store misbehavior (retains envelopes despite advertised deletion, denies delivery to specific recipients, becomes unavailable mid-share). | Documented in the threat model. Confidentiality of `dk[Vault]` remains protected by end-to-end encryption regardless. Availability mitigations: dealer revocation and re-share; manual `mv-dk-vault-raw-v1:` import path as a fallback for users whose store interaction fails. Honest-deletion is an operator obligation surfaced explicitly in [Impact](#impact). |
| Pending counter overflow on a receive-only account that never authorizes rollover. | The wallet displays "pending — accept incoming funds" whenever `pending > 0` and surfaces a stronger warning as the counter approaches the protocol limit. The user remains in charge of authorization. |
| Phishing via dApp-supplied derivation parameters. | Excluded by construction: the wallet accepts only the 32-byte FA metadata address from the dApp; every other derivation input is wallet-fixed. The wallet adapter exposes no generic "sign arbitrary bytes for confidential assets" hook. |

## Security Considerations

**Network/protocol impact.** This MIP does not change any cryptographic primitive verified on chain and adds no new Move modules. All multisig CA bootstrapping is off-chain; the chain sees only the pre-existing CA module surface plus a multisig's own `ek` registrations and confidential balance state.

**dApp ↔ wallet boundary.** The `ca_*` interface enforces in code that `dk` bytes never cross the boundary: the loader returns `dk` by `(accountAddress, tokenMetaAddr)`, proof routines accept one `dk` and one token address, balances are decrypted before crossing the boundary, and the wallet adapter ships no generic byte-signing or HKDF-parameter hook. A compromised dApp can request a CA operation the user did not intend, but cannot extract `dk` or build proofs that bind to a different `sender` than the user's wallet account (the wallet refuses `{ sender: <other>, mode: "submit" }`).

**Wallet-process compromise.** During a CA operation the loaded `dk[token]` resides in process memory. An attacker who compromises the wallet process during that window can decrypt that token's balance and construct valid CA proofs against `ek[token]`. Such proofs still require an Ed25519 / device / keyless-authenticated transaction to execute, so **funds remain safe**; the loss is confined to privacy for the tokens whose `dk` was loaded during the compromise window. Per-asset isolation contains the scope of the compromise for single-owner backings. For multisig backings, `dk[Vault]` is the broader privacy boundary: a wallet-process compromise that captures `dk[Vault]` exposes every asset under that vault, but unrelated vaults and the owner's personal accounts remain isolated. The mnemonic / pepper sits in memory under equivalent exposure rules; for hardware backings, the mnemonic is never present.

**Off-chain envelope store trust model.** The multisig vault application that hosts the envelope store is trusted for honest deletion of envelopes once recipients have acknowledged, availability during the share window, and non-discriminatory delivery to addressed recipients. It is **not** trusted for confidentiality: `dk[Vault]` is end-to-end encrypted under per-recipient X25519 + AES-GCM keys derived from on-chain Ed25519 owner pubkeys, and the store sees only ciphertext. It is **not** trusted for fund safety: every fund-moving multisig transaction still requires k-of-n on-chain Ed25519 approvals. A store that misbehaves — retains envelopes despite acknowledgement, leaks ciphertext to third parties — does not directly recover `dk[Vault]`, but it expands the population of parties who could decrypt if a recipient's Ed25519 owner key is later compromised. Owner key rotation hygiene is the ongoing user-side mitigation; dealer revocation and re-share, plus a manual `mv-dk-vault-raw-v1:` import path, are the operational fallbacks when the store itself fails.

**Auditor inclusion.** The wallet refuses to construct a confidential transfer when `get_chain_auditor()` returns `None`. This is the central inclusion check; conforming wallets MUST NOT silently omit the global auditor. The per-asset auditor is included whenever configured; the user sees the full auditor set in the transfer review.

**Tests** for the security-relevant paths are listed in [Testing](#testing): derivation byte-vector tests in the SDK, envelope round-trip and AAD-mismatch tests, lifecycle tests for `dk` and `dk[Vault]` zeroing on lock, and envelope-store acknowledgement and access-control tests.

**Cryptography references.** Twisted ElGamal and the Bulletproofs range-proof construction used by the CA protocol are documented in the on-chain `confidential_asset` module and the `@moveindustries/confidential-assets` package; this MIP composes them without modification. HKDF-SHA512 follows RFC 5869.

## Future Potential

- **Threshold ElGamal multi-owner custody.** A future protocol change that adds a threshold-ElGamal-aware Move verifier and threshold-friendly Bulletproofs/Sigma protocols would let k owners jointly hold a share of `dk` and produce one proof through MPC. Privacy would survive any single-owner wallet compromise. This MIP's shared-`dk[Vault]` design is the construction that is shippable today; threshold custody is a natural follow-on.
- **Post-quantum envelope.** The envelope's embedded version tag `mv-dk-vault-v1` exists so a successor envelope format can be introduced under a post-quantum KEM when one is ecosystem-ready.
- **Auditor registries.** The MIP consumes `get_chain_auditor()` and `get_asset_auditor(token)` views and surfaces them to the user. A future MIP could define richer governance, rotation, and discovery for auditor keys — and a per-account auditor list — without touching this MIP's wallet ↔ dApp interface.
- **Wallet-standard `movement:*` migration.** Today this MIP advertises CA support under both `aptos:confidentialAssets` and `movement:confidentialAssets`. A future ecosystem-coordinated release should drop the `aptos:*` aliases across the entire wallet-standard feature surface; this MIP just adopts the existing dual-publish convention rather than moving ahead of it.
- **In five years.** Movement has a stable, multi-wallet `ca_*` ecosystem in which confidential assets are the default for transfers above some user-configurable threshold, the per-asset auditor surface is widely adopted by regulated issuers, and multisig + CA is in mainstream treasury use without bespoke per-team tooling.

## Timeline

### Suggested implementation timeline

- **Phase 1 — SDK changes (4–6 weeks).** Land the four required changes in `@moveindustries/confidential-assets`: remove or restrict the auto-rollover helpers, add the build-only API, export canonical derivation helpers and `fromUniformBytes`, and ship the per-vault envelope `sealVaultDk` / `openVaultDk` helpers. Ship test vectors that pin every derivation and envelope byte layout.
- **Phase 2 — Multisig vault application (in parallel, 2–3 weeks).** Add the per-vault envelope store endpoint with per-recipient acknowledgement and deletion semantics.
- **Phase 3 — Wallet conformance (in parallel + 4–6 weeks after Phase 1).** Motion Wallet `feat/confidential-assets` branch reaches conformance with this MIP, including the canonical loader contract, multisig wallet entry model, `dk[Vault]` share / fetch / decrypt / import flows, and explicit "Accept incoming funds" UX. Third-party Movement wallets can be onboarded once Phase 1 and Phase 2 are released.
- **Phase 4 — dApp adoption (rolling).** Reference dApps (`gmove-multisig`, any first-party CA demo) integrate `useConfidentialAssets()`. Document A1–A6 conformance for third-party dApps.

### Suggested developer platform support timeline

- **SDK (`@moveindustries/confidential-assets`, `@moveindustries/ts-sdk`):** Phase 1.
- **Wallet adapter (`@moveindustries/wallet-adapter-react`):** Concurrent with Phase 1; adapter is a thin wrapper and only needs the dual-publish feature key and the `useConfidentialAssets` hook.
- **CLI:** None planned. The CLI is not a wallet trust boundary; users who need scripted CA operations use the SDK directly in a trusted environment.
- **Indexer:** No new requirements. `dk[Vault]` discovery is served by the multisig vault application's envelope store directly.

### Suggested deployment timeline

- **Devnet:** SDK, wallet, and multisig vault application envelope store wired up end-to-end on devnet within the implementation window.
- **Testnet:** After SDK, wallet, and envelope store all reach conformance and pass the rehearsals in [Testing](#testing).
- **Mainnet:** Targeted immediately after testnet conformance and successful rehearsal. The MIP author will refresh this estimate after the gatekeeper design review per the template's guidance.

## Open Questions

These are the design decisions the spec has not yet made. Each must be resolved before any wallet implementation writes CA registrations on chain under that decision, because once `ek` is registered the choice in effect at registration time is the only one that reproduces a matching `dk` thereafter.

| # | Question | Options | Notes |
|---|---|---|---|
| 1 | **Per-transfer auditor address UX** | (a) Per-transfer entry only. (b) Wallet-managed address book. (c) dApp provides a list, wallet confirms. | Global and per-asset auditors are out of scope here; this concerns only optional per-transfer (voluntary) auditors. For v1, (a) or (c) is likely sufficient. |
| 2 | **Spam-token rollover and surfacing** | How does the wallet display unsolicited inbound tokens, and how is rollover scoped? | Suggested answer: per-token rollover only (no "accept all"), display unknown tokens with a warning badge (not hidden, not blocked), no allowlist dependency. Avoids gas-extraction traps and keeps spam filtering out of the critical path while leaving room for an allowlist-based enhancement later. |
| 3 | **Ephemeral-key expiry mid-proof (keyless)** | If the keyless ephemeral key expires between proof construction and submission, does the wallet (a) silently trigger keyless re-auth and re-sign the existing proof, or (b) surface a dedicated error and ask the user to retry? | The proof itself binds to `senderAddress` via Fiat–Shamir, not to the ephemeral key, so the proof survives re-auth and can be wrapped in a freshly-signed transaction. Affects perceived reliability for sessions held open near the ephemeral-key expiry boundary. |