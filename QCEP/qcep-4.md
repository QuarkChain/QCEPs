---
qcep: 4
title: Staking contract standard for PoSW mining rewards
author: @junjiah
status: Draft
created: 2019-10-29
---

## Simple Summary

We would like to propose a standard interface and reference implementation for staking contracts where stakers and miners can work together in a mutually beneficial fashion, where stakers can provide funds to lower the mining difficulty for shard-miners.

## Specification

The contract has two parties: stakers (potentially **many**), and **one** miner. Once deployed, the miner should use the contract address as the coinbase.

The incentive model has two folds:

1. stakers are willing to provide the funds to earn interests (which depends on the hash power of the miner);
1. the miner will experience higher effective hash power since PoSW will lower the block difficulty (which depends on the amount of stakes).

## Implementation

The contract has following main APIs:

- fallback payable for adding stakes
- `withdrawStakes(amount)` for stakers to withdraw (including profit)
- `withdrawFee` for miners to get mining fees
- `updateMiner` for miners to update address

During the mining process, block rewards will be sent to this contract, and counted as interests for stakers. Part of the whole profit (`minerFee`) will be distributed to the miner, while others will be proportionally distributed to stakers according to the amount of stakes.

Following is a reference implementation of the staking contract.

```
pragma solidity ^0.5.1;


contract StakingPool {

    struct StakerInfo {
        uint128 stakes;
        uint128 arrPos;
    }

    mapping (address => StakerInfo) public stakerInfo;
    address[] public stakers;
    uint256 public totalStakes;
    address payable public miner;
    uint256 public feeRateBp;
    uint256 public minerFee;
    uint128 public maxStakers;

    constructor(address payable _miner, uint256 _feeRateBp, uint128 _maxStakers) public {
        require(_feeRateBp <= 10000, "Fee rate should be in basis point.");
        miner = _miner;
        feeRateBp = _feeRateBp;
        maxStakers = _maxStakers;
    }

    function getDividend() public view returns (uint256) {
        return address(this).balance - totalStakes;
    }

    // Add stakes
    function () external payable {
        calculatePayout();

        StakerInfo storage info = stakerInfo[msg.sender];
        // New staker
        if (info.stakes == 0) {
            require(stakers.length < maxStakers, "Too many stakers.");
            info.arrPos = uint128(stakers.length);
            stakers.push(msg.sender);
        }

        info.stakes += uint128(msg.value);
        totalStakes += msg.value;
        require(totalStakes >= msg.value, "Addition overflow.");
    }

    function withdrawStakes(uint128 amount) public {
        StakerInfo storage info = stakerInfo[msg.sender];
        require(info.stakes >= amount, "Should have enough stakes to withdraw.");
        info.stakes -= amount;
        totalStakes -= amount;

        msg.sender.transfer(amount);

        if (info.stakes == 0) {
            stakerInfo[stakers[stakers.length - 1]].arrPos = info.arrPos;
            stakers[info.arrPos] = stakers[stakers.length - 1];
            stakers.length--;
            delete stakerInfo[msg.sender];
        }

        calculatePayout();
    }

    function withdrawFee() public {
        require(msg.sender == miner, "Only miner can withdraw fees.");
        calculatePayout();
        uint256 toWithdraw = minerFee;
        minerFee = 0;
        msg.sender.transfer(toWithdraw);
    }

    function updateMiner(address payable _miner) public {
        require(msg.sender == miner, "Only miner can update the address.");
        calculatePayout();
        miner = _miner;
    }

    function calculatePayout() private {
        uint256 dividend = getDividend();
        uint256 stakerPayout = dividend * (10000 - feeRateBp) / 10000;
        uint256 paid = 0;
        for (uint16 i = 0; i < stakers.length; i++) {
            StakerInfo storage info = stakerInfo[stakers[i]];
            uint256 toPay = info.stakes * stakerPayout / totalStakes;
            paid += toPay;
            info.stakes += uint128(toPay);
        }

        totalStakes += paid;
        require(totalStakes >= paid, "Addition overflow.");

        minerFee += dividend - paid;

        require(address(this).balance >= totalStakes, "Balance should be more than stakes.");
    }
}
```

## Implications

Note that due to the lock-up mechanism of PoSW, stakers / miners may not be able to withdraw the funds if mining is going on.

Thus stakers and the miner should reach social consensus to temporarily stop using the contract address for mining if either side needs to withdraw the funds.

## Copyright
Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
