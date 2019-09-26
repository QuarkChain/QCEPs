---
qcep: 3
title: Precompiled Contracts for interacting with Multi-Native-Token
author: @hanyunx
status: Draft
created: 2019-09-20
---

## Simple Summary
Precompiled contracts for interacting with multi-native-token are required in order for developers to conduct multi-native-token-related operations.

## Specification
Add precompiled contracts for querying current token ID and creating messages with different transfer token ID.

Address of  `MNT`: 0x514b430001 \
Address of `APPLY_MNT`: 0x514b430002

### Querying Current Token ID
The precompiled contract requires no input and returns current message’s token ID if there is enough gas.

### Transfer Multi-Native-Token
```
Input: (address, transfer_token_ID, value, data(Optional))
Output: If the length of the input is less than 96 bytes or there is not enough gas, the call fails.
        Otherwise, create a new message with specified token ID and apply this message.
```
In the input, the first three parameters are all 32-byte big-endian numbers which take up 96 bytes in total. So the length of the input should be at least 96 bytes and the remaining part will be encoded as message data.

Note that the EVM will reject the contract call if the given `transfer_token_id` is non default and this token-id is never queried.

### Gas Costs
- Gas cost for `MNT`: 3
- Gas cost for `APPLY_MNT`: 3

## Backwards Compatibility
As with the introduction of any precompiled contract, contracts that already use the given addresses will change their semantics. 

## Implementation
In `MNT`, the implementation will checkout the gas of current message and return the message’s instance variable `transfer_token_id` as queried.
The following contract will invoke the precompiled contract and trigger the querying of current token ID:
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
In `APPLY_MNT`, The length of input data should be no less than 96 bytes, where the target code address , token id,  and value are all 32-byte-long and the rest part of input should be the message data.
After encoding the input, a new message can be created and applied. \
The function `applyMNT` in the following contract is an example of invoking the precompiled contract `APPLY_MNT`. In this example, when `APPLY_MNT` is executed, a new message with the given token id will be created and applied. Since the message data is set as the encoded `setTokenId()`, function `setTokenId()` will be called.
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

    function applyMNT( uint256 _tokenId) public {
        uint256[4] memory input;
        uint p;
        input[0] = uint256(address(this));
        input[1] = _tokenId;
        input[2] = 0;

        bytes memory data;
        data = abi.encodeWithSignature("setTokenId()");

        assembly{
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
Note that EVM will reject the contract call if the given `transfer_token_id` is non default  and never queried. In the implementation, an instance variable `token_id_queried` is created and set to `False` by default in class `Message`. It will be updated to `True` only when `MNT` is called, that is, the token id in involved message is queried. In `APPLY_MNT`, the newly created message will be applied in the end, where `token_id_queried` is checked to make sure the non-default transfer token id has been queried. If not, it will return 0 which indicates the failure of contract call:
```
    if (
        res == 1
        and code != b""
        and msg.transfer_token_id != ext.default_state_token
        and not msg.token_id_queried
        and msg.value != 0
    ):
        res = 0
```

## Copyright
Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
