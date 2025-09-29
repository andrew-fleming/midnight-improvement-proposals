---
MIP: ? # assigned by editors
Title: Contract-based Fungible Token Standard
Authors: Andrew Fleming <@andrew-fleming>
Status: Proposed
Category: Standards
Created: 2025-09-25
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
The latter point will be available when contract-to-contract interactions are fully supported.

Furthermore, this standard provides wallets and off-chain services a formal way to access token metadata.

## Specification

### Ledger

All ledger fields _must_ be exported with the current MRC standard as a prefix followed by two underscores (using `MRC20__` as a placeholder).
All ledger fields _must_ use double underscores after the standard name `MRC20__balances`.

#### MRC20__name

The immutable token name.

```ts
sealed ledger MRC20__name: Opaque<"string">
```

#### MRC20__symbol

The immutable token symbol.

```ts
sealed ledger MRC20__symbol: Opaque<"string">
```

#### MRC20__decimals

The immutable token decimals.

```ts
sealed ledger MRC20__decimals: u8
```

#### MRC20__balances

Mapping from account addresses to their token balances.

```ts
ledger MRC20__balances: Map<Either<ZswapCoinPublicKey, ContractAddress>, Uint<128>>
```

#### MRC20__allowances

Mapping from owner accounts to spender accounts and their allowances.

```ts
ledger MRC20__allowances: Map<Either<ZswapCoinPublicKey, ContractAddress>, Map<Either<ZswapCoinPublicKey, ContractAddress>, Uint<128>>>
```

#### MRC20__totalSupply

The total token supply.

```ts
ledger MRC20__totalSupply: Uint<128>
```

### Circuits

#### totalSupply

Returns the value of tokens in existence.

```ts
circuit totalSupply() → Uint<128>
```

#### balanceOf

Returns the value of tokens owned by `account`.

```ts
circuit balanceOf(account: Either<ZswapCoinPublicKey, ContractAddress>) → Uint<128>
```

#### transfer

Moves a `value` amount of tokens from the caller’s account to `to`.

```ts
circuit transfer(to: Either<ZswapCoinPublicKey, ContractAddress>, value: Uint<128>) → Boolean
```

#### allowance

Returns the amount which `spender` is still allowed to spend from `owner`.

```ts
circuit allowance(
    owner: Either<ZswapCoinPublicKey, ContractAddress>,
    spender: Either<ZswapCoinPublicKey, ContractAddress>
  ) → Uint<128>
```

#### approve

Sets a `value` amount of tokens as allowance of `spender` over the caller’s tokens.

```ts
circuit approve(spender: Either<ZswapCoinPublicKey, ContractAddress>, value: Uint<128>) → Boolean
```

#### transferFrom

Moves `value` tokens from `from` to `to` using the allowance mechanism.
`value` is the deducted from the caller’s allowance.

```ts
circuit transferFrom(
    from: Either<ZswapCoinPublicKey, ContractAddress>,
    to: Either<ZswapCoinPublicKey, ContractAddress>,
    value: Uint<128>
  ) → Boolean
```

## Rationale

Explain the design decisions behind the proposed change. Why was this particular approach chosen? What alternatives were considered, and why were they rejected?

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

- **Dedicated `Storage` object**: Contracts use export a ledger API interface which enforces the token interface i.e.

```ts
struct IMRC20_Storage {
  balances: Map<Either<ZswapCoinPublicKey, ContractAddress>, Uint<128>>
  name: Opaque<"string">
  ...
}

export ledger Storage: IMRC20_Storage;
```

Struct fields cannot be ADT types.

This is an interesting option though as it could easily enable abstraction strategies and improve dx such as config files and/or decorators.

### Circuits API

The circuits API provides an approximation of the EIP20 API for cross-chain compatibility.
The main differences reside in the input and output types:

- `address` => `Either<ZswapCoinPublicKey, ContractAddress>`
- `u256` => `Uint<128>`
- `string` => `Opaque<"string">`

## Path to Active

What does it mean to get from Accepted to Active, and how this will be achieved.

### Acceptance Criteria

Explain what objective milestones need to be achieved in order for the MIP to achieve Active status.

### Implementation Plan

Describe how the MIP will be put into practice.

## Backwards Compatibility Assessment

Describe how the proposed change affects existing systems, applications, and users. Will it require a hard fork? Are there any compatibility issues? How will they be addressed?

## Security Considerations

Analyze the potential security implications of the proposed change. Are there any new attack vectors or vulnerabilities introduced? How will they be mitigated?

## Implementation

Describe how the proposed change will be implemented. Which parts/components of the Midnight need to be modified? What are the dependencies, if any?

## Testing

Describe the testing procedures for the proposed change. What tests will be performed to ensure that it works as expected and does not introduce any regressions?

## References (Optional)

Are there any external sources that are referenced in this document, or that add to the efficacy of this MIP?

## Acknowledgements

List the contributors that were not the Authors, this will include any workshop participants.

## Copyright Waiver

All contributions (code and text) submitted in this MIP must be licensed under the Apache License, Version 2.0. Submission requires agreement to the Midnight Foundation Contributor License Agreement [Link to CLA], which includes the assignment of copyright for your contributions to the Foundation.
