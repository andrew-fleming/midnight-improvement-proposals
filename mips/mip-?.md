<!--
 Copyright 2025 Midnight Foundation

 Licensed under the Apache License, Version 2.0 (the "License");
 you may not use this file except in compliance with the License.
 You may obtain a copy of the License at

     https://www.apache.org/licenses/LICENSE-2.0

 Unless required by applicable law or agreed to in writing, software
 distributed under the License is distributed on an "AS IS" BASIS,
 WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 See the License for the specific language governing permissions and
 limitations under the License.
-->

---
MIP: ? # assigned by editors
Title: Contract-based Fungible Token Standard
Authors: Andrew Fleming <@andrew-fleming>
Status: Proposed
Category: Standards
Created: 2025-10-02
Requires: [None]
Replaces: [None]
License: Apache-2.0
---

## Abstract

A standard for contract-based fungible tokens on Midnight.

Inspired by [EIP-20](https://eips.ethereum.org/EIPS/eip-20).

## Motivation

The following standard allows for the implementation of a standard API for unshielded contract-based tokens
(as opposed to ledger-based UTXOs) within smart contracts.

This standard provides basic functionality to transfer tokens,
as well as allow tokens to be approved so they can be spent by another on-chain third party.
The latter point will be more relevant when contract-to-contract interactions are fully supported.

Furthermore, this standard provides wallets and off-chain services a formal way to access token metadata.

## Specification

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://www.rfc-editor.org/rfc/rfc2119) and [RFC 8174](https://www.rfc-editor.org/rfc/rfc8174).

### Ledger

All ledger fields _must_ be exported with the current MRC standard as a prefix followed by two underscores (using `MRC__` as a placeholder).
All ledger fields _must_ use double underscores after the standard name `MRC__balances`.
Additional token-specific ledger fields SHOULD follow the same namespacing pattern
(e.g., `MRC__customField`) to prevent conflicts and ensure forward compatibility.

#### `MRC__name`

The immutable token name.

```ts
sealed ledger MRC__name: Opaque<"string">
```

#### `MRC__symbol`

The immutable token symbol.

```ts
sealed ledger MRC__symbol: Opaque<"string">
```

#### `MRC__decimals`

The immutable token decimals.

```ts
sealed ledger MRC__decimals: u8
```

#### `MRC__balances`

Mapping from account addresses to their token balances.

```ts
ledger MRC__balances: Map<Either<ZswapCoinPublicKey, ContractAddress>, Uint<256>>
```

#### `MRC__allowances`

Mapping from owner accounts to spender accounts and their allowances.

```ts
ledger MRC__allowances: Map<Either<ZswapCoinPublicKey, ContractAddress>, Map<Either<ZswapCoinPublicKey, ContractAddress>, Uint<256>>>
```

#### `MRC__totalSupply`

The total token supply.

```ts
ledger MRC__totalSupply: Uint<256>
```

### Circuits

#### `totalSupply`

Returns the value of tokens in existence.

```ts
circuit totalSupply() → Uint<256>
```

#### `balanceOf`

Returns the value of tokens owned by `account`.

```ts
circuit balanceOf(account: Either<ZswapCoinPublicKey, ContractAddress>) → Uint<256>
```

#### `transfer`

Moves a `value` amount of tokens from the caller’s account to `to`.
Calling this circuit must emit a Transfer event (even if the transfer amount is `0`).

The circuit MUST fail if the sender's balance is insufficient.

If the `to` address is equal to the zero address (either `ZswapCoinPublicKey` or `ContractAddress`),
the circuit MUST fail.

```ts
circuit transfer(to: Either<ZswapCoinPublicKey, ContractAddress>, value: Uint<256>) → []
```

#### `allowance`

Returns the amount which `spender` is still allowed to spend from `owner`.

```ts
circuit allowance(
    owner: Either<ZswapCoinPublicKey, ContractAddress>,
    spender: Either<ZswapCoinPublicKey, ContractAddress>
  ) → Uint<256>
```

#### `approve`

Sets a `value` amount of tokens as allowance of `spender` over the caller’s tokens.
The allowance must be set to `0` when setting another `value` for the same `spender`.

> NOTE: Enforcing the allowance to be `0` avoids the race condition issue of EIP20.
To better address this issue, approvals can be atomic.
While this standard does not implement atomic approvals directly,
implementers may extend this spec with optional support for signature-based approvals or callback-based patterns to enable atomic `approve` + `transferFrom` flows once c2c is available.

```ts
circuit approve(spender: Either<ZswapCoinPublicKey, ContractAddress>, value: Uint<256>) → []
```

#### `transferFrom`

Moves `value` tokens from `from` to `to` using the allowance mechanism.
`value` is deducted from the allowance previously approved to the caller by the `from` address.
Calling this circuit must emit a Transfer event (even if the transfer amount is `0`).

The `transferFrom` circuit allows another user or contract (almost always a contract) to spend the tokens of `from`.
The circuit MUST fail unless the `from` account has deliberately authorized the caller to spend at least `value` amount of tokens.

If the `from` or `to` address is equal to the zero address (either `ZswapCoinPublicKey` or `ContractAddress`),
the circuit MUST fail.

```ts
circuit transferFrom(
    from: Either<ZswapCoinPublicKey, ContractAddress>,
    to: Either<ZswapCoinPublicKey, ContractAddress>,
    value: Uint<256>
  ) → []
```

### Events

> NOTE: The `event` keyword and event emission are not currently supported in Compact.
This specification defines event formats to guide future tooling and contract upgrades.

#### `Transfer`

The Transfer event MUST trigger when tokens are ever moved (including zero value transfers).
Tokens that are minted and burned should also emit this event.

```ts
// pseudo code
// event emission is not yet supported
event Transfer(
  indexed from: Either<ZswapCoinPublicKey, ContractAddress>,
  indexed to: Either<ZswapCoinPublicKey, ContractAddress>,
  indexed value: Uint<256>
)
```

#### `Approval`

The Approval event MUST trigger when the allowance of a spender is set.
`value` MUST be the new allowance.

```ts
// pseudo code
// event emission is not yet supported
event Approval(
  indexed owner: Either<ZswapCoinPublicKey, ContractAddress>,
  indexed spender: Either<ZswapCoinPublicKey, ContractAddress>,
  indexed value: Uint<256>
)
```

## Rationale

### Ledger API

In many other blockchain ecosystems, getter calls are "free"—meaning a wallet can query a user's balance by calling `ERC20.balanceOf(target)`.
Midnight's architecture requires a different approach because circuit calls to a contract require a generated proof.
It's infeasible for off-chain services to produce a proof for every metadata query; therefore,
this standard works within the architecture of Midnight by leveraging the contract ledger for token metadata.
The contract ledger API utilizes a prefix pattern that standardizes how these fields can be found for wallets,
explorers, and other off-chain services.

#### Other Considered API patterns

- **Special character prefix**: The use of `$` or other special character domains i.e. `$balances`.
This may cause semantic conflicts with other languages.

- **Dedicated `Storage` object**: Contracts export a ledger API interface which enforces the required fields i.e.

```ts
struct IMRC_Storage {
  balances: Map<Either<ZswapCoinPublicKey, ContractAddress>, Uint<256>>
  name: Opaque<"string">
  ...
}

export ledger Storage: IMRC_Storage;
```

Struct fields cannot be ADT types.

This is an interesting option though as it could easily enable abstraction strategies and improve dx such as config files and/or decorators.

### Circuits API

The circuits API provides an approximation of the EIP20 API for cross-chain compatibility.
The main differences reside in the input and output types:

- `address` => `Either<ZswapCoinPublicKey, ContractAddress>`
- `string` => `Opaque<"string">`

### Disallowed zero address transfers

Transfers to or from zero addresses are explicitly disallowed to prevent accidental token loss and ambiguous behavior.
It is up to the implementing contract to define minting or burning functionality,
including whether such behavior is supported at all and whether publicly callable circuits are provided.

### Midnight native UTXOs

The current functionality of UTXOs does not support expressive functionality that many token protocols require
such as freezing or pausing assets as well as whitelisting and blacklisting users.
Such functionality requires a means of enforcing logic within the UTXO spend itself (custom spend logic).

### Callbacks

Callbacks are very useful for introspection and crafting atomic `approve` + `transferFrom` transactions.
Contract-to-contract (c2c) interactions are not yet fully supported in the Midnight network.

### Signature-based approvals

As a token standard, the technical specifications should favor both a permissible and minimal interface.
Therefore, this standard does not enforce signature-based approvals.

## Backwards Compatibility Assessment

The proposed changes do not introduce breaking changes.
As the first formalized token standard for the Midnight ecosystem,
this MIP does not break compatibility with existing systems.
There are no deployed contracts that would be affected by this specification.

Contract-based token contracts (prior to the standard) deployed before this standard's adoption will continue to function normally; however,
they may not be recognized by compliant wallets, explorers, or other ecosystem tools.

## Security Considerations

The well-documented ERC20 race conditions with the `approve` + `transferFrom` flow are mitigated in this standard by forcing approvals to only occur when starting with an allowance of `0`.
Further improvements to address this issue,
and to remove the potential extra transaction of setting approvals to `0`,
are recommended with an approval signature or introducing callbacks.

## Implementation

OpenZeppelin's Contracts for Compact library will provide an implementation when all features are available.
The following implementation approximates this standard:

- [OpenZeppelin implementation](https://github.com/OpenZeppelin/compact-contracts/blob/main/contracts/src/token/FungibleToken.compact)

## Testing

This MIP defines an interface specification rather than a concrete implementation.
Testing focuses on validating that implementations comply with the standard and integrate properly with ecosystem tools.

### Compliance Testing

Token implementations claiming standard compliance SHOULD be validated against:

1. **Interface Compliance**: Required circuits are present and accept correct parameters.
2. **State Variable Naming**: Mandatory ledger variables follow the specified naming conventions (e.g., `MRC__balances`).
3. **Metadata Exposure**: Metadata fields are properly exposed and queryable.
4. **Event Conformance**: `Transfer` and `Approval` events are emitted with correct parameters and indexed fields.

### Integration Testing

Implementations SHOULD verify compatibility with ecosystem infrastructure:

- **Wallets**: Automatic token detection and balance display.
- **Block Explorers**: Token identification and transaction history formatting.
- **Bridges**: Cross-chain transfer functionality (if applicable).

### Reference Implementation

The reference implementation will include a comprehensive test suite demonstrating standard compliance.
Token creators are encouraged to use this as a baseline for their own testing.

## References (Optional)

Are there any external sources that are referenced in this document, or that add to the efficacy of this MIP?

- [EIP20](https://eips.ethereum.org/EIPS/eip-20)

## Future Considerations

### Signature-based approvals (similar to EIP-2612)

Allow token holders to approve a spender via a signed message,
without requiring an explicit on-chain approval transaction.
This improves UX and reduces Dust cost.

This pattern is a strong candidate for a dedicated extension MIP once message standards stabilize.

### Callback Support (C2C Interactions)

Once contract-to-contract (C2C) calls are fully supported,
more advanced token flows become possible.

### Off-Chain Metadata: `MRC__contractURI`

Several ecosystems (e.g. Ethereum via [EIP-7572](https://eips.ethereum.org/EIPS/eip-7572)) expose a `contractURI()` function to return a URI pointing to structured off-chain metadata.

To ensure lean and bridge-compatible token definitions, this MRC does not include `MRC__contractURI` in the base interface.
However, ecosystem tools may expect or support it.
A follow-up MIP may define a standard metadata schema and hosting guidelines.

## Copyright Waiver

All contributions (code and text) submitted in this MIP must be licensed under the Apache License, Version 2.0. Submission requires agreement to the Midnight Foundation Contributor License Agreement [Link to CLA], which includes the assignment of copyright for your contributions to the Foundation.
