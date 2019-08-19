---
qcep: 1
title: PoSW staking for mining root chain
author: @qizhou, @junjiah
status: Draft
created: 2019-08-16
---

## Simple Summary

To allow staking for root chain mining, we propose the use of a smart contract on chain 0 shard 0 for locking / unlocking the stakes.

## Specification

The spec has two parts, one is about the smart contract and how the user interacts with it, the other one is how the cluster / light client interacts with the contract when producing / validating the blockchain.

### Spec: Staking Contract

```
contract RootChainPoSWStaking {

    // 3 day locking period.
    uint public constant LOCK_DURATION = 24 * 3600 * 3;

    struct Stake {
        bool unlocked;
        uint256 withdrawableTimestamp;
        uint256 amount;
    }

    mapping (address => Stake) public stakes;

    function () external payable {
        Stake storage stake = stakes[msg.sender];
        require(!stake.unlocked, "should only stake more in locked state");

        uint256 newAmount = stake.amount + msg.value;
        require(newAmount > stake.amount, "addition overflow");
        stake.amount = newAmount;
    }

    function lock() public {
        Stake storage stake = stakes[msg.sender];
        require(stake.unlocked, "should not lock already-locked accounts");

        stake.unlocked = false;
    }

    function unlock() public {
        Stake storage stake = stakes[msg.sender];
        require(!stake.unlocked, "should not unlock already-unlocked accounts");
        require(stake.amount > 0, "should have existing stakes");

        stake.unlocked = true;
        stake.withdrawableTimestamp = now + LOCK_DURATION;
    }

    function withdraw(uint256 amount) public {
        Stake storage stake = stakes[msg.sender];
        require(stake.unlocked && now >= stake.withdrawableTimestamp);
        require(amount <= stake.amount);

        stake.amount -= amount;

        msg.sender.transfer(amount);
    }

    function withdrawAll() public {
        Stake memory stake = stakes[msg.sender];
        require(stake.amount > 0);
        withdraw(stake.amount);
    }

    // Used by root chain for determining stakes.
    function getLockedStakes(address staker) public view returns (uint256) {
        Stake memory stake = stakes[staker];
        if (stake.unlocked) {
            return 0;
        }
        return stake.amount;
    }

}
```

This contract provides the root chain an access to determine the miner's stakes if the root chain miner willingly *locks* those stakes to participate the PoSW consensus and enjoy the potentially lower difficulty.

For each staker, there are two states in the contract: **locked** and **unlocked**. Only when the staker is in the locked state, his/her stakes will be taken into account when used as a coinbase address to produce a root block.

When the staker is in the locked state (default state):

1. the staker can increase the stakes by directly sending QKC to the contract, and will be seen by the root chain once the including minor block is included by the root chain;
1. the staker can **unlock** the stakes, which will be seen by the root chain as forfeiting the potential PoSW benefits.

When the staker is in the unlocked state:

1. once unlocked, the staker needs to wait 3 days to withdraw the funds;
1. the staker can again turn into **locked** state, and if he/she wants to unlock again, the withdrawable timestamp will get delayed accordingly;
1. the staker can't add stakes when unlocked.

### Spec: Cluster Interaction with the Contract

The root chain keeps track of chain 0 shard 0's blockchain, and when it's creating a new block, it will use that shard's latest minor block **up until the current root block** as the state to evaluate the coinbase address's stakes, no matter the producing root block includes that shard's latest header or not.

Then root chain will make an intra-cluster RPC call to chain 0 shard 0 as `getRootChainStakes` with the recipient and minor block hash as the arguments and get back the stakes:

```python
class GetRootChainStakesRequest:
  FIELDS = [
    ("address", Address),
    ("minor_block_hash", hash256),
  ]

class GetRootChainStakesResponse:
  FIELDS = [
    ("error_code", uint32),
    ("stakes", biguint),
  ]
```

Then it will compare the stakes with the PoSW configuration and determine whether the root chain miner should have lower block difficulty.

Note that for all other shards (not chain 0 shard 0), this RPC should be a no-op and return None.

## Backwards Compatibility

This will be a breaking change and needs a hard fork, together with enabling EVM aimed at end of September.

## Implications

For future light clients who are only interested in root chain, they will be required to have full history of chain 0 shard 0 as well (or has a trusted source for querying stakes).

## Implementation

One implementation-specific question is how the staking contract will be deployed and baked into the root chain consensus.

A simple way is to use `CREATE2` with a pre-determined contract creator address and init code, so the contract address will be known in advance. Then, when chain 0 shard 0 checks the stakes, it will return 0 if the contract is not yet deployed.

In this way, after EVM is enabled, even if the staking contract is not immediately available, consensus will not be broken.

## Copyright
Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
