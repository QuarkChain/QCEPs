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
pragma solidity >0.4.99 <0.6.0;
contract RootChainPoSWStaking {

    // 3 day locking period.
    uint public constant LOCK_DURATION = 24 * 3600 * 3;

    struct Stake {
        bool unlocked;
        uint256 withdrawableTimestamp;
        uint256 amount;
        address signer;
    }

    mapping (address => Stake) public stakes;

    function addStakes(Stake storage stake, uint256 amount) private {
        if (amount > 0) {
            uint256 newAmount = stake.amount + amount;
            require(newAmount > stake.amount, "addition overflow");
            stake.amount = newAmount;
        }
    }

    function setSigner(address signer) external payable {
        Stake storage stake = stakes[msg.sender];
        require(!stake.unlocked, "should only set signer in locked state");

        stake.signer = signer;
        addStakes(stake, msg.value);
    }

    function () external payable {
        Stake storage stake = stakes[msg.sender];
        require(!stake.unlocked, "should only add stakes in locked state");

        addStakes(stake, msg.value);

    }

    function lock() public payable {
        Stake storage stake = stakes[msg.sender];
        require(stake.unlocked, "should not lock already-locked accounts");

        stake.unlocked = false;
        addStakes(stake, msg.value);
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
    function getLockedStakes(address staker) public view returns (uint256, address) {
        Stake memory stake = stakes[staker];
        if (stake.unlocked) {
            return (0, address(0));
        }

        address signer;
        if (stake.signer == address(0)) {
            signer = staker;
        } else {
            signer = stake.signer;
        }
        return (stake.amount, signer);
    }

}
```

This contract gives the root chain an access to determine the miner's stakes if the miner willingly *locks* those stakes to participate the PoSW consensus and have the potentially lower difficulty.

For each staker, there are two states in the contract: **locked** and **unlocked**. Only when the staker is in the locked state, his/her stakes will be taken into account when used as a coinbase address to produce a root block. To make sure no malicious miner can impersonate an address with large amounts of tokens, we also allow staker to provide a hot-wallet `signer` address, and will require the mined root block to have a signature matching the recorded address in the contract. If such address is not provided, it will default to the staker's address. This mechanism is similar to our current [proof-of-guardian](https://github.com/QuarkChain/pyquarkchain/wiki/Ethash-with-Guardian) to ensure the safety of mainnet during the early stage. The signature in the root block header is signed on the header hash excluding `nonce`, `mixhash` and `signature` itself, as seen in [`RootBlockHeader.sign_with_private_key`](https://github.com/QuarkChain/pyquarkchain/blob/829fbf0eea57a2b48719e1cac66befc69378adf9/quarkchain/core.py#L961-L964) method.

When the staker is in the locked state (default state):

1. the staker can increase the stakes by directly sending QKC to the contract, and will be seen by the root chain once the including minor block is included by the root chain;
1. the staker can set the signer address that is used to sign the submitted root block, with potentially more stakes (the function is `payable`);
1. the staker can **unlock** the stakes, which will be seen by the root chain as forfeiting the potential PoSW benefits.

When the staker is in the unlocked state:

1. once unlocked, the staker needs to wait 3 days to withdraw the funds;
1. the staker can again turn into **locked** state (with potentially more stakes, as the function is `payable`), and if he/she wants to unlock again, the withdrawable timestamp will get delayed accordingly;
1. the staker can't add stakes when unlocked.

### Spec: Cluster Interaction with the Contract

The root chain keeps track of chain 0 shard 0's blockchain, and when it's creating a new block, it will use that shard's latest minor block **up until the current root block** as the state to evaluate the coinbase address's stakes, no matter the producing root block includes that shard's latest header or not.

Then root chain will make an intra-cluster RPC call to chain 0 shard 0 as `getRootChainStakes` with the recipient and minor block hash as the arguments and get back the stakes along with recorded signer address:

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
    ("signer", FixedSizeBytesSerializer(20)),
  ]
```

The node will then verify `RootBlockHeader.signature` with the returned signer address (note: NOT the coinbase address which points to the staker, but the one from the RPC call), and only proceed PoSW if matches, otherwise treat it as a normal root block work submission.

Then it will compare the stakes with the PoSW configuration and determine whether the root chain miner should have lower block difficulty.

Note that for all other shards (not chain 0 shard 0), this RPC should return an error.

## Backwards Compatibility

This will be a breaking change and needs a hard fork, together with enabling EVM aimed at the end of September. It will go parallel with existing proof-of-guardian mechanism.

## Implications

For future light clients who are only interested in root chain, they will be required to have full history of chain 0 shard 0 as well (or has a trusted source for querying stakes).

## Implementation

One implementation-specific question is how the staking contract will be deployed and baked into the root chain consensus.

A simple way is to use `CREATE2` with a pre-determined contract creator address and init code, so the contract address will be known in advance. Then, when chain 0 shard 0 checks the stakes, it will return 0 if the contract is not yet deployed.

In this way, after EVM is enabled, even if the staking contract is not immediately available, consensus will not break.

A pre-compiled contract is implemented solely to deploy the staking contract ([QuarkChain/pyquarkchain#725](https://github.com/QuarkChain/pyquarkchain/pull/725)), which will deploy to the predetermined address.

The following contract will invoke the pre-compiled contract and trigger the staking contract deployment:

```
pragma solidity >=0.4.22 <0.6.0;

contract Contract {

    address public addr;

    function () external {
        bytes20[1] memory res;
        assembly {
            if iszero(call(not(0), 0x514b430003, 0, 0, 0x00, res, 0x20)) {
                revert(0, 0)
            }
        }
        addr = address(uint160(res[0]));
    }
}
```

Note pre-compiled contract can only succeed once.

Finally the deployed contract address will be `0x514b43000000000000000000000000000000000100000001` (chain 0 shard 0).

## Appendix

`RootChainPoSWStaking` contract bytecode, with compiler version `0.5.11+commit.c082d0b4` targeting at EVM version `petersburg` and optimizer turned on for 200 runs:

0x608060405234801561001057600080fd5b50610700806100206000396000f3fe60806040526004361061007b5760003560e01c8063853828b61161004e578063853828b6146101b5578063a69df4b5146101ca578063f83d08ba146101df578063fd8c4646146101e75761007b565b806316934fc4146100d85780632e1a7d4d1461013c578063485d3834146101685780636c19e7831461018f575b336000908152602081905260409020805460ff16156100cb5760405162461bcd60e51b815260040180806020018281038252602681526020018061062e6026913960400191505060405180910390fd5b6100d5813461023b565b50005b3480156100e457600080fd5b5061010b600480360360208110156100fb57600080fd5b50356001600160a01b031661029b565b6040805194151585526020850193909352838301919091526001600160a01b03166060830152519081900360800190f35b34801561014857600080fd5b506101666004803603602081101561015f57600080fd5b50356102cf565b005b34801561017457600080fd5b5061017d61034a565b60408051918252519081900360200190f35b610166600480360360208110156101a557600080fd5b50356001600160a01b0316610351565b3480156101c157600080fd5b506101666103c8565b3480156101d657600080fd5b50610166610436565b6101666104f7565b3480156101f357600080fd5b5061021a6004803603602081101561020a57600080fd5b50356001600160a01b0316610558565b604080519283526001600160a01b0390911660208301528051918290030190f35b8015610297576002820154808201908111610291576040805162461bcd60e51b81526020600482015260116024820152706164646974696f6e206f766572666c6f7760781b604482015290519081900360640190fd5b60028301555b5050565b600060208190529081526040902080546001820154600283015460039093015460ff9092169290916001600160a01b031684565b336000908152602081905260409020805460ff1680156102f3575080600101544210155b6102fc57600080fd5b806002015482111561030d57600080fd5b6002810180548390039055604051339083156108fc029084906000818181858888f19350505050158015610345573d6000803e3d6000fd5b505050565b6203f48081565b336000908152602081905260409020805460ff16156103a15760405162461bcd60e51b81526004018080602001828103825260268152602001806106546026913960400191505060405180910390fd5b6003810180546001600160a01b0319166001600160a01b038416179055610297813461023b565b6103d06105fa565b5033600090815260208181526040918290208251608081018452815460ff16151581526001820154928101929092526002810154928201839052600301546001600160a01b031660608201529061042657600080fd5b61043381604001516102cf565b50565b336000908152602081905260409020805460ff16156104865760405162461bcd60e51b815260040180806020018281038252602b8152602001806106a1602b913960400191505060405180910390fd5b60008160020154116104df576040805162461bcd60e51b815260206004820152601b60248201527f73686f756c642068617665206578697374696e67207374616b65730000000000604482015290519081900360640190fd5b805460ff191660019081178255426203f48001910155565b336000908152602081905260409020805460ff166105465760405162461bcd60e51b815260040180806020018281038252602781526020018061067a6027913960400191505060405180910390fd5b805460ff19168155610433813461023b565b6000806105636105fa565b506001600160a01b03808416600090815260208181526040918290208251608081018452815460ff161580158252600183015493820193909352600282015493810193909352600301549092166060820152906105c75750600091508190506105f5565b60608101516000906001600160a01b03166105e35750836105ea565b5060608101515b604090910151925090505b915091565b6040518060800160405280600015158152602001600081526020016000815260200160006001600160a01b03168152509056fe73686f756c64206f6e6c7920616464207374616b657320696e206c6f636b656420737461746573686f756c64206f6e6c7920736574207369676e657220696e206c6f636b656420737461746573686f756c64206e6f74206c6f636b20616c72656164792d6c6f636b6564206163636f756e747373686f756c64206e6f7420756e6c6f636b20616c72656164792d756e6c6f636b6564206163636f756e7473a265627a7a72315820f2c044ad50ee08e7e49c575b49e8de27cac8322afdb97780b779aa1af44e40d364736f6c634300050b0032

## Copyright
Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
