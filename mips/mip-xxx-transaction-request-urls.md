---
mip:
title: Transaction Request URLs (`movement:`)
author: ganymedio
Status: Draft
type: Standard (Interface)
created: 2026-07-28
---

# MIP-X - Transaction Request URLs (`movement:`)

## Summary

A `movement:` URL scheme for payment requests. A request names a recipient address, an asset, and an amount; a conforming wallet resolves it, selects the entry function, simulates, and presents a confirmation screen. The format is deliverable over any channel that carries a string — QR code, NFC tag, hyperlink, chat message — and every conforming wallet parses it identically. This solves the absence of a portable payment-request representation on Movement: today every merchant integration, invoice, or tap-to-pay flow must target one specific wallet's bespoke deep-link format.

### Out of scope

Non-payment transaction requests (the `call-`, `sign-`, and `bcs-` form prefixes are reserved, not defined), authenticated requests (`sig`/`signer` reserved), fee sponsorship (`fee_payer` reserved), name resolution (`name` reserved), and any on-chain memo or reference mechanism — no framework transfer function carries one, and this MIP does not add one.

## High-level Overview

A request is a URL: scheme, form prefix stating the intent, recipient address, chain ID, query parameters.

```
movement:pay-0x9b21…c4f2@126?amount=1250000000&label=Coffee
movement:pay-0x9b21…c4f2@126?asset=0x52ab…77e1&amount=25000000
movement:pay-0x9b21…c4f2@126?amount=12.5e8&expires=1785600000&message=Invoice%204471
```

(Addresses elided here for readability; real requests carry all 64 hex characters.)

Every parameter is a suggestion the payer may change. The wallet — not the URL — selects the entry function, simulates the transaction, and renders a confirmation screen, which is what the user authorizes.

## Impact

- **Wallets:** register the scheme, implement the parser, processing pipeline, and confirmation requirements below.
- **Request producers** (merchants, invoicing, point-of-sale, NFC personalization): emit conforming URLs.
- **SDK:** ship a shared parser/builder and conformance vectors.
- **Chain, node operators, asset issuers, existing users:** no action; nothing here is on-chain.

Without a standard, payment acceptance is rebuilt per wallet and physical-world acceptance stays blocked on wallet-specific integrations. No dependencies on other MIPs.

## Alternative Solutions

- **Name the entry function (or a BCS payload) in the URL.** Rejected: it pins the dispatch path at authoring time, so a URL stamped on a physical card breaks when the correct path changes — the in-progress coin-to-fungible-asset migration is exactly such a change — and an opaque payload is not human-inspectable.
- **Nominal-unit amounts.** Rejected: they require trusting a decimals value at authoring time; atomic units plus the on-chain cross-check remove that trust.
- **Symbols or legacy coin types for `asset`.** Rejected: the FA metadata object address is the one identity every fungible asset has; symbols are not unique, and coin types do not exist for FA-native assets.
- **Short-form addresses.** Rejected: with no checksum, short forms make many corruptions still-valid addresses.
- **A companion `https://` universal-link form.** Rejected: it would place a hostname operator, who observes every scan and tap, in the middle of a payment flow. Custom schemes are squattable by any app, so "the wallet opened" is evidence of nothing and the security model rests entirely on the confirmation screen. Movement SHOULD register `movement` as a provisional scheme per [RFC 7595](https://www.rfc-editor.org/rfc/rfc7595).
- **In-page wallet-standard only, no URL format.** Rejected: it requires a live JavaScript context with a connected wallet; a QR code, NFC tag, or chat link has neither. The two surfaces are complementary.

## Specification and Implementation Details

The key words MUST, MUST NOT, SHOULD, SHOULD NOT, and MAY are to be interpreted as described in [RFC 2119](https://www.rfc-editor.org/rfc/rfc2119).

### Syntax

The grammar is ABNF ([RFC 5234](https://www.rfc-editor.org/rfc/rfc5234)):

```abnf
request         = "movement:" form [ "@" chain-id ] [ "?" parameters ]

form            = "pay-" long-address       ; recipient; long form only

long-address    = "0x" 64HEXDIGL
special-address = "0x" HEXDIGL                  ; 0x0-0xf; `asset` only
HEXDIGL         = DIGIT / "a" / "b" / "c" / "d" / "e" / "f"

asset-value     = long-address / special-address
chain-id        = 1*3DIGIT                      ; 0-255; ChainId is a u8

parameters      = parameter *( "&" parameter )
parameter       = key "=" value
key             = "amount"  / "asset"  / "expires"
                / "label"   / "message" / "decimals" / ext-key
ext-key         = "x-" 1*( ALPHA / DIGIT / "-" / "_" )

amount-value    = uint / sci
uint            = 1*20DIGIT
sci             = mantissa ( "e" / "E" ) 1*2DIGIT
mantissa        = 1*DIGIT [ "." 1*DIGIT ]
```

Lexical rules:

- Hex is **lowercase**. A wallet MUST reject uppercase or mixed-case hex rather than normalize it: Movement addresses carry no checksum, and accepting mixed case invites the false inference that they do.
- Percent-encoding is [RFC 3986](https://www.rfc-editor.org/rfc/rfc3986). Inside `label`, `message`, and `x-` values, the characters `&` `=` `#` `%` `+` MUST be percent-encoded. `+` MUST NOT be used to mean a space.
- A duplicated parameter key is a rejection. Parameter order is not significant.
- An unknown form prefix, an unknown key outside the `x-` space, a forbidden or reserved key, or a malformed value is a rejection with a user-visible reason — never a silently dropped field. This MIP defines `pay-` and reserves `call-`, `sign-`, and `bcs-`; a future incompatible revision takes a new prefix.

### Target address

Mandatory; the payment recipient. MUST be a `long-address`. A `special-address` in the target position is rejected — `0x0`–`0xf` are framework addresses, never recipients.

### Chain ID

Producers MUST include a chain ID. Movement mainnet is `126`; Movement testnet (Bardock) is `250`. A wallet MUST reject a request whose chain ID differs from its configured network and MUST NOT switch networks to satisfy a request — the 8-bit chain ID space is shared with every other Move network. A request without a chain ID targets the configured network, stated on the confirmation screen.

### `amount`

The quantity to transfer, in the **atomic unit** of the asset (octas for MOVE: `1250000000` is 12.5 MOVE). It MUST evaluate to an exact non-negative integer ≤ `u64::MAX`; anything out of range, fractional after applying the exponent, or carrying a sign is a rejection — never saturate, wrap, or round. Scientific notation is permitted (`12.5e8` is exactly `1250000000`); the exponent SHOULD be the asset's decimal count. If absent, the wallet prompts the payer. Implementations MUST use exact decimal or big-integer arithmetic: a double cannot represent `u64::MAX`.

### `asset`

The fungible asset to transfer, as its **FA metadata object address** (MOVE's is `0xa`). If absent, the asset is MOVE. The wallet verifies on chain that a fungible asset exists at the address and renders only its on-chain symbol and name. Legacy coin **type** strings (`0x1::aptos_coin::AptosCoin`) MUST NOT be used: they require escaping in URLs, do not exist for FA-native assets, and push the coin-versus-FA dispatch decision onto the producer. A wallet MAY enforce a curated asset policy; such a list is not part of this format.

### `expires`

Optional; a Unix timestamp in seconds. A wallet MUST reject a request whose `expires` is past, and MUST set the transaction's `expiration_timestamp_secs` to no later than `expires`. Because that field is part of the signed `RawTransaction`, the expiry is VM-enforced, not UI advice.

### `label` and `message`

Optional, percent-encoded UTF-8 display strings: `label` names the payee, `message` describes the payment. Both are untrusted (see [Confirmation screen](#confirmation-screen)) and never reach the chain — no framework transfer function carries a memo, and this MIP does not invent one.

### `decimals`

Optional; a producer-stated cross-check on the asset's decimal count. The wallet MUST read the authoritative value from `0x1::fungible_asset::Metadata` and MUST reject on mismatch — a mismatch is a corrupted request or an attempt to misrepresent the amount's scale.

### Forbidden and reserved keys

`value`, `gas`, `gasLimit`, `gasPrice`, `max_gas_amount`, `gas_unit_price`, and `sequence_number` MUST cause rejection: `value` is a foreign amount key whose silent loss would mean a wrong-amount payment; gas and sequence number are wallet-owned. `sig`, `signer`, `fee_payer`, `name`, and `nonce` are reserved for future MIPs and MUST also be rejected. Producers needing private parameters use the `x-` space; wallets ignore unrecognized `x-` keys, which MUST NOT affect the transaction.

### Entry-function selection

The URL expresses intent — recipient, asset, amount — never a function to call. The wallet selects the entry function at payment time:

1. **MOVE:** `0x1::aptos_account::transfer(address, u64)`.
2. **A paired asset where the payer holds a residual `CoinStore<C>` balance:** `0x1::aptos_account::transfer_coins<C>(address, u64)`, which spans both the coin store and the FA store in one transaction. `C` comes from `coin::paired_coin(metadata)`; the residual balance is detected by reading `0x1::coin::CoinStore<C>` at the payer's address.
3. **Every other asset:** `0x1::aptos_account::transfer_fungible_assets(Object<Metadata>, address, u64)`. It MUST NOT be selected when rule 2 applies — it has no `CoinStore` fallback and fails on a residual legacy balance that `coin::balance` reports as spendable.

`aptos_account` functions are used because they create the recipient's account when it does not exist. Wallets MUST NOT assume either branch of the `OPERATIONS_DEFAULT_TO_FA_APT_STORE` feature gate — it can flip in a framework release. Migration of a residual `CoinStore` is never required inside a payment.

### Wallet processing

Normative, in order; any failing step aborts the request with a user-visible reason, and the user is never shown a confirmation screen for a request that failed a step.

1. Parse against the grammar; reject unknown, forbidden, reserved, duplicate, and malformed input.
2. Reject if `expires` is past; reject on chain-ID mismatch.
3. Read `0x1::fungible_asset::Metadata` at the asset address; reject if absent; cross-check `decimals`.
4. Apply the object-address check (below) to the target.
5. Resolve pairing and balance location; select the entry function.
6. Check the payer's MOVE balance covers the network's minimum transaction gas cost; render an "insufficient MOVE for network fees" state when it does not, rather than surfacing `MAX_GAS_UNITS_BELOW_MIN_TRANSACTION_GAS_UNITS` from simulation.
7. Build the transaction (wallet-chosen gas and sequence number; expiration ≤ `expires`) and simulate. A transfer can abort for reasons the producer cannot observe — a dispatchable withdraw/deposit hook, a frozen store, a recipient who opted out of direct coin transfers. Render an abort as a typed failure, never a confirmable transaction.
8. Render the confirmation screen from the **simulated** result; on confirmation, sign and submit.

A wallet without node access MUST reject the request: every check above is a chain read, and a wallet that cannot reach a node cannot submit either.

#### Object-address check

Object and account addresses are indistinguishable as strings, and a deposit to an object address generally cannot be spent: the auto-created account's `authentication_key` is the address itself. The wallet MUST read the target's resources and (1) **hard-reject** if the target hosts `0x1::fungible_asset::Metadata` or `FungibleStore` — asset-infrastructure objects are never valid recipients; (2) require an **explicit typed acknowledgement** if `0x1::object::ObjectCore` is otherwise present — object-owned treasuries with an `ExtendRef` custodian are legitimate payees and cannot be distinguished by inspection. The check MUST test for the presence of `ObjectCore`, never the absence of `0x1::account::Account`, which object addresses acquire as a side effect of the mistake being guarded against. Absence of all these resources is the common case — a never-used address — and MUST NOT be an error.

#### Confirmation screen

The confirmation screen, not the URL, is what the payer authorizes. Wallets MUST:

1. Render the recipient address in full — 64 hex characters, never elided. With no address checksum, the middle of an address is where a substitution hides.
2. Render the amount in nominal units with the symbol and name from **on-chain metadata**, alongside the atomic value, and the metadata address for a non-default asset. Nothing whose provenance is a URL parameter may be rendered as though it were chain state.
3. Render `label` and `message` in a visually distinct, clearly untrusted region; cap rendered length and truncate visibly; strip or escape bidirectional-override and zero-width code points; never interpret them as markup or links; never let them occupy the region where recipient, asset, amount, or network render.
4. Render the network by name; keep `amount` and recipient editable — every value in a request is a suggestion.
5. Never auto-submit. No configuration, allowlist, or trusted-producer flag may bypass confirmation.

### Transport bindings

- **QR codes / links / chat:** encode the URL as-is. On Android the intent filter MUST declare only the scheme (`<data android:scheme="movement" />`): request URIs are opaque, and a filter adding `host` or `path` never matches and fails silently. Browser-extension wallets, which cannot register a scheme handler, MAY intercept `movement:` link activation via a content script.
- **NFC:** a single NDEF URI record (TNF `0x01`, type `U`, identifier code `0x00` — no abbreviation). A device presenting a request dynamically MUST emulate an NFC Forum Type 4 tag carrying the same record. A MOVE request (~101 bytes) fits an NTAG213; a request naming a 64-hex `asset` (~174 bytes) needs NTAG215 or larger.

### Reconciliation

No parameter of this format reaches the chain, so a producer cannot tag a payment — it matches by a unique receiving address per request, or by exact amount within the VM-enforced `expires` window on a shared address. Producers MUST NOT rely on `label`, `message`, or `x-` keys for reconciliation.

## Reference Implementation

- `@moveindustries/ts-sdk`: `parseTransactionRequestUrl` / `buildTransactionRequestUrl` plus a versioned conformance-vector suite.
- Wallet: [MoveIndustries/motion-wallet#132](https://github.com/MoveIndustries/motion-wallet/pull/132) — parser with exact `bigint` amounts and typed rejections, content-script link interception.

No feature flag; nothing is consensus-visible. A request sent to a wallet without support simply does not open.

## Testing

Conformance vectors (grammar accept/reject pairs with expected rejection reasons, `amount` boundary cases around `u64::MAX`, the `0xa` object-address fixture) are versioned with this MIP and CI-gate the SDK. Wallet suites cover asset resolution across all three selection rules, object-address handling, chain-ID and expiry behavior, and display-string hardening. Two independent implementations passing the same vectors gate leaving Draft.

## Risks and Drawbacks

There is no incumbent standard, so backward compatibility is unaffected; existing wallet-specific deep links keep working under their own schemes. Forward compatibility is the principal risk — a URL stamped on a card cannot be revised — and is carried by the fail-closed rules: unknown-key rejection plus reserved prefixes and keys let future revisions land without a v1 wallet misreading a v2 request, and wallet-side dispatch absorbs framework changes in wallet releases rather than reprinted cards. Accepted drawbacks: a non-MOVE request does not fit the smallest NFC tags, and requests are unauthenticated.

## Security Considerations

A request is attacker-controlled — links, QR codes, and tags are unauthenticated, and the scheme handler is unowned — so the parser is an untrusted-input boundary. The preserved properties: the payer can only sign what was rendered; no parameter reaches the chain unrendered; nothing irreversible happens without confirmation. Substitution of an entire request by an attacker controlling the delivery channel is not defended against; the confirmation-screen requirements are the only defence, which is why they are normative. A broadly published request links every payment made against its address.

## Future Potential

The reserved keys and prefixes stage the roadmap without a grammar v2: authenticated requests (`sig`/`signer`) making swapped tags detectable, fee sponsorship (`fee_payer`) so a payer with no MOVE can pay in a stablecoin, name resolution (`name`) once a canonical registry exists, one-shot requests (`nonce`), and generic call forms (`call-`, `bcs-`). The end state is one format under all Movement payment acceptance — cards, tags, terminals — with authentication and sponsorship layered on top.

## Timeline

Phase 1: SDK parser, builder, and vectors (2–3 weeks). Phase 2: wallet pipeline and confirmation screen (3–4 weeks). Phase 3: transport bindings validated on physical tags (1–2 weeks, overlapping). Phase 4: a second implementation runs the vectors; Draft → Last Call is gated on that result, not a date. Nothing deploys on chain.

## Open Questions

1. **Companion `https://` universal-link form?** Drafted as no (see Alternative Solutions); wallet distribution experience should decide.
2. **Accept legacy coin-type input for `asset` and normalize?** Drafted as reject; depends on how much unmigrated coin-only supply persists.
3. **Should the object-address acknowledgement gate be a hard rejection?** Wrongly rejecting an object-owned treasury costs a failed payment; wrongly accepting a store object costs the funds. Needs data on how common object treasuries are.
