---
qcep: 6
title: System Contract for General Native Token Management
author: @junjiah
status: Draft
created: 2019-12-06
---

## Simple Summary
This QCEP proposes using a system contract to provide utility values for native tokens. The system contract will be deployed on every shard, allowing users to propose exchange rates between the token and QKC so the token can be used to pay transaction fees.


## Specification
In each shard, any user can interact with the general native token manager system contract to propose an exchange rate that will be used to convert the native tokens to the corresponding amount of QKC to pay for transaction fees, for any transaction using that token as gas (`gas_token_id`).

The exchange rate proposal process is similar to an English auction: for any token, the highest exchange rate will be applied, while the proposer (hereinafter referred to as *admin*) should also deposit a certain amount of QKC as the gas reserve. When executing the transaction, the blockchain will query the exchange rate from the system contract and deduct the QKC reserve accordingly.


### Parameters
Following parameters should be set by the supervisor:

- Minimum amount of QKC as gas reserve: once the admin’s QKC reserve drops below this amount, any new exchange rate proposal would take over the current one.
- Minimum amount of QKC to start proposing the exchange rate: each proposal’s corresponding QKC balance should be higher than this value.

Some administrative flags:

- Supervisor: an account responsible for early-stage administrative works such as setting parameters
- Require token registration: whether to require token being registered before proposing exchange rates and functioning as gas

### Gas Reserve Exchange Rate Proposal Process
1. Deploy the general token management system contract with reasonable parameters.
1. To make a native token available for paying gas, register through the contract’s `registerToken` method with that particular native token.
1. Anyone can propose an exchange rate for a token and provide liquidity with QKC as gas reserves (greater than `minGasReserveInit`).
1. The highest exchange rate proposer will be the token admin, in the meantime the QKC gas reserve will be locked to provide liquidity. The other proposers can withdraw QKC at any time.
1. Transactions using the aforementioned native tokens will automatically convert to QKC (for miners), while deducting the gas reserve from the contract and adding native token balances in the contract for the token admin. The admin can a) withdraw those native tokens at any time; b) deposit more QKC as gas reserve.
1. Other people can become admins by either providing a higher exchange rate, or wait until the current admin’s gas reserves drops below `minGasReserveMaintain` and then anyone can outbid with arbitrary exchange rates.

Note that step 5 happens in QuarkChain’s consensus, essentially it means when using native token to pay gas a) the transaction sender’s native token will be transferred to the system contract, under the record of the token admin; b) corresponding amount of QKC will be deducted from the contract and be consumed by the miner.

### Refund
- To prevent excessive arbitrage, the highest exchange rate proposer can set the refund percentage which is between 10% and 100%, which mandates the amount of converted QKC for refund if the transaction doesn’t use up all the gas. The default value is 50%. The rest will be burnt to zero address.
- For a cross-shard transaction, the refund percentage only depends on the value in the source shard.

## API Methods

- [registerToken](#registertoken)
- [proposeNewExchangeRate](#proposenewexchangerate)
- [depositGasReserve](#depositgasreserve)
- [setRefundPercentage](#setrefundpercentage)
- [withdrawGasReserve](#withdrawgasreserve)
- [withdrawNativeToken](#withdrawnativetoken)
- [calculateGasPrice](#calculategasprice)

Note that `payAsGas` can only be called by the consensus as it requires the message sender to be the contract itself.

### registerToken

Before the token can be used to pay transaction fees, it should be registered by invoking this method with the particular token as the transfer token ID.

#### Parameters

None. But `msg.value` (i.e. the transfer token ID) needs to be the native token.

#### Returns

None

### proposeNewExchangeRate

Proposing a new exchange rate the a native token.

#### Parameters

1. `tokenId`, uint128 - token ID for paying transaction fees
1. `rateNumerator`, uint128 - exchange rate numerator
1. `rateDenominator`, uint128 - exchange rate denominator

#### Returns

None

### depositGasReserve

Add more QKC to the reserve for the native token. The sender doesn’t need to be the admin.

#### Parameters

1. `tokenId`, uint128 - token ID for paying transaction fees

#### Returns

None

### setRefundPercentage

Token admin can adjust the refund rate.

#### Parameters

1. `tokenId`, uint128 - token ID for the desired refund rate
1. `refundPercentage`, uin64 - refund rate in percent, between 10 and 100

#### Returns

None

### withdrawGasReserve

Withdraw gas reserve, if and only if the sender is not the current admin.

#### Parameters

1. `tokenId`, uint128 - token ID for withdrawing

#### Returns

None

### withdrawNativeToken

Withdraw native tokens, which is accumulated by either registration or converting for QKC as gas.

#### Parameters

1. `tokenId`, uint128 - token ID for withdrawing

#### Returns

None

### calculateGasPrice

Calculate converted QKC gas price given a native token and target gas price in native token.

#### Parameters

1. `tokenId`, uint128 - token ID for gas price calculation
1. `gasPrice`, uint128 - gas price in native token

#### Returns

1. `refund percentage` 
1. `convertedGasPrice` in QKC
1. `admin` for the native token

## Copyright
Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).


