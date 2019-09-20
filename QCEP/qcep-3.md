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

Address of  `current_mnt_id`: 0x514b430001
Address of `transfer_mnt`: 0x514b430002

### Querying Current Token ID
The precompiled contract requires no input and returns current message’s token ID if there is enough gas.

### Transfer Multi-Native-Token
```
Input: (address, transfer_token_ID, value, data)
Output: If the length of the input is less than 96 bytes or there is not enough gas, the call fails.
        Otherwise, create a new message with specified token ID and apply this message.
```
In the input, the first three parameters are all 32-byte big-endian numbers which take up 96 bytes in total. So the length of the input should be at least 96 bytes and the remaining part will be encoded as message data.

Note that the EVM will reject the contract call if the given transfer_token_id is non default and this token-id is never queried.

### Gas Costs
- Gas cost for `current_mnt_id`: 3
- Gas cost for `transfer_mnt`: 3

## Rationale
The rationale.

## Backwards Compatibility
As with the introduction of any precompiled contract, contracts that already use the given addresses will change their semantics. 

## Test Cases
test for `current_mnt_id`:
- not enough gas
- normal case

## Implementation
In `current_mnt_id`, the implementation will checkout the gas of current message and return the message’s instance variable `transfer_token_id` as queried.
The following contract will invoke the precompiled contract and trigger the querying of current token ID:
```
pragma solidity >=0.4.22 <0.6.0;
contract Contract {
    uint256 private tokenId;
    
    function get_cur_token_id() public returns (uint256[1] memory out) {
        uint256[2] memory x;
        assembly {
            let y := call(100, 0x514b430001, 0, x, 0x40, out, 0x20)
            switch y case 0 {
                revert(0, 0)
            }
        }
        tokenId = out[0];
    }
}
```

In `transfer_mnt`, The length of input data should be no less than 96 bytes, where the target code address , token id,  and value are all 32-byte-long and the rest part of input should be the message data.
After encoding the input, a new message can be created and applied.
The following contract Caller will invoke the precompiled contract and trigger the transference of multi-native-token.
The last input `callF`, namely the message data as mentioned above, is a flag to choose whether to call function `f()` or `g()` otherwise. In this way, we can tell if message data is correctly  passed by checking whether the correct function in contract Callee is called as expected.

```
pragma solidity >=0.4.22 <0.6.0;

contract Callee {
    uint256 public state = 0;
    function f() public payable {
        // make sure token ID queried
        uint256[1] memory id;
        assembly {
            let _ := call(not(0), 0x514b430001, 0, 0, 0x00, id, 0x40)
        }
        state = 1;
    }
    
    function g() public payable {
        // make sure token ID queried
        uint256[1] memory id;
        assembly {
            let _ := call(not(0), 0x514b430001, 0, 0, 0x00, id, 0x40)
        }
        state = 2;
    }
}

contract Caller {
    function transfer_mnt(uint256 addr, uint256 tokenId, uint256 value, bool callF) public returns(uint p) {
        uint256[4] memory input;
        input[0] = addr;
        input[1] = tokenId;
        input[2] = value;
        
        bytes memory data;
        
        if (callF) {
            data = abi.encodeWithSignature("f()");
        }
        else {
            data = abi.encodeWithSignature("g()");
        }
        
        uint256 x;
        assembly {
            x := mload(add(data, 0x20))
        }
        
        input[3] = x;
        
        assembly{
            if iszero(call(not(0), 0x514b430002, 0, input, 0x80, p, 0x20)){
                revert(0, 0)
            }
        }
    }
}
```

Note that EVM will reject the contract call if the given `transfer_token_id` is non default  and never queried. In the implementation, an instance variable `token_id_queried` is created and set to `False` by default in class `Message`. It will be updated to `True` only if `current_mnt_id` is called, that is, the token id in involved message is queried. In `transfer_mnt`, the newly created message will be applied in the end, where `token_id_queried` will be checked to make sure the non-default transfer token id has been queried. If not, it will return 0 which indicates the failure of contract call:
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
