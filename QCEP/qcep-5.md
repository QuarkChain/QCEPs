---
qcep: 5
title: System Contract for Non-Reserved Native Token Management
author: @hanyunx, @junjiah
status: Draft
created: 2019-11-26
---

## Simple Summary

This QCEP proposes a mechanism for users to auction / issue native tokens by interacting with a system contract.

## Specification

Following roles are involved in the native token management system:

- **Supervisor**: the one who sets the auction parameters, initiates the auction process and has access to pausing and resuming the auction. Supervisorship can be transferred.

- **Token Owners**: winners of token auctions, who then have the ownership of the native tokens and can start minting. Ownership can be transferred. Note that the owner can renounce the ownership by transferring it to a zero address.
- **Bidders**: anyone taking part in the auction by placing.

### Auction Type

Participants make increasingly higher bids, and the bidder with the highest price wins the auction, on the condition that the token ID is valid (more on this later).

### Auction Parameters

The following parameter has been set in the system contract:

- **Overtime period**: if a successful bid is placed within 5 minutes of the scheduled closing time of an auction, then the auction will go into overtime.

Other auction parameters will be set at the very beginning of the system contract by the supervisor, which are

- **Minimum bid price**: the smallest amount that can be bid by a user.
- **Minimum increment in percentage**: the minimum amount in percentage a bid can be increased by.
- **Duration**: the period for one round of auction.

Note these parameters can be updated later on.

### New Token Bidding Process

1. After the system contract becomes live (i.e. deployed), the supervisor is required to set auction parameters and resume the auction at the beginning.

1. Each round of auction is started when the first valid bidding for the current round is placed, and will end after `duration` time from this point unless automatic extension happens.

1. Bidders place their bids with the price they would like to pay, the token ID they want and the target round number. One bidder can bid multiple times. The `value` they set in a transaction can be taken as the bidding deposit which will be accumulated in the system. For example, if a user places a bid with `value` of 5 QKC in round 2 but does not win in this round, and later places another bid with `value` of 10 QKC in round 4, the balance will be 15 QKC available in total, which means the bidding price can be as high as 15 QKC.

    To make a successful bid, the bid price should be no less than the minimum bid price set by the supervisor, and should be higher than the current highest bid price meeting the minimum increment requirement.

1. One round of auction can be settled in two ways after the end time. One is to be explicitly settled by any user, or, it will be settled automatically when someone places a valid bid for the next round.

1. After the auction is settled, the winner becomes the owner of the new token. The equivalent amount of QKC with the bidding price will be burnt from the owner’s bidding balance, while the surplus bidding balance can be either withdrawn or used for future auctions. The owner can mint the auctioned new token, transfer ownership, etc.

### Withdrawing bidding deposit

- Bidders who are not the highest bidder may be able to withdraw their bidding deposit at anytime.

- Highest bidder can withdraw spare deposit (if any) after current auction ends, i.e. the total bidding balance minus the bidding price.

### Non-reserved token and whitelist of reserved token

This auction system is applicable for non-reserved tokens, whose names can be a mix of capital letters and numbers with length between 5 and 12. Token ID, the unique identifier of a token in the auction system, is the decimal format of token name which can be taken as a numeral system with base number of 36 (`0`-`9` followed by `A`-`Z`) . As a result, the auctioned token ID should be larger than 1727603 which is the decimal format of `ZZZZ`.

As an exception, the supervisor can whitelist some reserved token IDs for auction.

## API Methods

- [whitelistTokenId](#whitelisttokenid)
- [setAuctionParams](#setauctionparams)
- [pauseAuction](#pauseauction)
- [resumeAuction](#resumeauction)
- [getAuctionState](#getauctionstate)
- [getNativeTokenInfo](#getnativetokeninfo)
- [isPaused](#ispaused)
- [bidNewToken](#bidnewtoken)
- [endAuction](#endauction)
- [mintNewToken](#mintnewtoken)
- [transferOwnership](#transferownership)
- [withdraw](#withdraw)

### whitelistTokenId

*Only* for the supervisor to set the whitelist for reserved tokens.

#### Parameters

1. `tokenId`, uint128 - ID of the token to be set
1. `whitelisted`, bool - true if the token ID should be available for auction, otherwise should be unavailable

#### Returns

None

### setAuctionParams

*Only* for the supervisor to set three auction parameters. No return value.

#### Parameters

1. `minPriceInQKC`, uint64 - minimum bid price in QKC
1. `minIncrementInPercent`, uint64 - minimum increment in percentage
1. `duration`, 64 uint - duration of a round of auction

#### Returns

None

### pauseAuction

*Only* for the supervisor to pause the auction. No one can bid if `pauseAuction` is called.

#### Parameters

None

#### Returns

None

### resumeAuction

*Only* for the supervisor to resume the auction. Users can continue to bid once. `resumeAuction` is called.

#### Parameters

None

#### Returns

None

### getAuctionState

Returns information of the highest bid at the time of query, current round number and the end time of this round of auction.

#### Parameters

None

#### Returns

1. `tokenId` - token ID proposed in the highest bid
1. `newTokenPrice` - bid price of the highest bid
1. `bidder` - address of the highest bidder
1. `round` - current round number of the auction
1. `endTime` - end time of this round of auction

### getNativeTokenInfo

Returns information related to queried token.

#### Parameters

1. `tokenId`, uint128 - token ID of the token to be queried

#### Returns

1. `createTime` - returns 0 if the queried token has not been auctioned, otherwise is the time when the token is created
1. `owner` - address of the owner of the queried token
1. `totalSupply` - total amount of the queried token that has been minted by the owner

### isPaused

Returns whether the auction is paused or ongoing.

#### Parameters

None

#### Returns

1. `isPaused` - returns `true` if the auction is paused, otherwise `false`

### bidNewToken

Users can place their bids by calling `bidNewToken` method as long as the auction is ongoing.

#### Parameters

1. `tokenId`, uint128 - ID of the token to be auctioned
1. `bidPrice`, uint128 - bid price one would like to offer for the token ID proposed
1. `round`, uint64 - number of target round of auction

#### Returns

None

### endAuction

By calling `endAuction`, users can settle a round of auction after its end time. The highest bid at the end time wins the bid and its bidder turns the owner of the auctioned token.

#### Parameters

None

#### Returns

None

### mintNewToken

Owners can mint their native tokens by calling `mintNewToken`. A token can be minted many times.

#### Parameters

1. `tokenId`, uint128 - ID of the token to be minted
1. `amount`, uint256 - amount of the token to be minted

#### Returns

None

### transferOwnership

Owners can transfer their ownership or renounce their ownership by free will by calling `transferOwnership`.

#### Parameters

1. `tokenId`, uint128 - ID of the token whose owner will be changed
1. `newOwner`, address - address of the new owner, ownership of the token is renounced if the address is 0

#### Returns
None

### withdraw

Users can withdraw all their balance remained in the auction system as long as they are not the highest bidder of the auction at the time of query.

#### Parameters

None

#### Returns

None

## Copyright
Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
