---
qcep: 3
title: Precompiled Contracts for interacting with Multi-Native-Token
author: @hanyunx
status: Final
created: 2019-09-20
---

## Simple Summary
Precompiled contracts for interacting with multi-native-token are required in order for developers to conduct multi-native-token-related operations.

## Specification
Add precompiled contracts for querying current multi-native-token ID (`MNT`) and applying new messages with different multi-native-token ID (`APPLY_MNT`).

- Address of `MNT`: 0x514b430001
- Address of `APPLY_MNT`: 0x514b430002

### Querying Current Token ID
The precompiled contract requires no input and returns current message’s token ID if there is enough gas.

### Transfer Multi-Native-Token
```
Input: (address, transfer_token_ID, value, data(Optional))
Output: If the length of the input is less than 96 bytes or there is not enough gas, the call fails.
        Otherwise, create a new message with specified token ID and apply this message.
```
In the input, the first three parameters are all 32-byte big-endian numbers which take up 96 bytes in total. So the length of the input should be at least 96 bytes and the remaining part will be encoded as message data.

Note that the EVM will reject the contract call if the given multi-native-token id is non-default and current token id is never queried.

### Gas Costs
- Gas cost for `MNT`: 3
- Gas cost for `APPLY_MNT`: 3

## Backwards Compatibility
As with the introduction of any precompiled contract, contracts that already use the given addresses will change their semantics.

## Implementation
In `MNT`, the implementation will return the message’s `tokenId` as queried.
The following contract will invoke the precompiled contract and trigger the querying of current multi-native-token ID:
```
pragma solidity >=0.4.22 <0.6.0;

contract Contract {
    uint256 private tokenId;

    function getCurrentTokenId() {
        uint256[1] memory out;
        assembly {
            if iszero(call(100, 0x514b430001, 0, 0, 0x00, out, 0x20)){
                revert(0, 0)
            }
        }
        tokenId = out[0];
    }
}
```

\
In `APPLY_MNT`, The length of input data should be no less than 96 bytes, where the target code address , token id, and value are all 32-byte-long and the rest part of input should be the message data.
After encoding the input, a new message can be created and applied. \
The function `applyMNT` in the following contract is an example of invoking the precompiled contract `APPLY_MNT`. In this example, when `APPLY_MNT` is executed, a new message with the given token id will be created and then applied. Since the message data is assigned as encoded `setTokenId()`, function `setTokenId` will be called when the new message is applied. Of particular note is `MNT` is called in this function to query current token id, where the queried result is precisely the token id passed to `applyMNT`. In this way, it can be guaranteed that the value of `_tokenId` passed to `applyMNT` has been queried before new message with this token id is sucessfully applied, which is essential to non-default token.
```
pragma solidity >=0.4.22 <0.6.0;

contract Contract {
    uint256 public tokenId;

    function setTokenId() public payable {
        // make sure token ID queried
        uint256[1] memory out;
        assembly {
            // call precompiled contract MNT
            if iszero(call(100, 0x514b430001, 0, 0, 0x00, out, 0x20)){
                revert(0, 0)
            }
        }
        tokenId = out[0];
    }

    function applyMNT(uint256 _tokenId) public {
        uint256[4] memory input;
        uint p;
        input[0] = uint256(address(this));
        input[1] = _tokenId;
        input[2] = 0;

        bytes memory data;
        data = abi.encodeWithSignature("setTokenId()");

        assembly {
            // input[3] = uint256(data)
            mstore(add(input, 0x60), mload(add(data, 0x20)))

            // call precompiled contract APPLY_MNT
            if iszero(call(not(0), 0x514b430002, 0, input, 0x80, p, 0x20)){
                revert(0, 0)
            }
        }
    }
}
```

\
It's very important to note that if the given `tokenId` is non-default **AND** current token id is never queried, EVM will **REJECT** the contract call.

## Copyright
Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
