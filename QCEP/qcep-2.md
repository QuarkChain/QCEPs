---
qcep: 2
title: Improved Qkchash with LLRB rotation statistics (QkchashX)
author: @junjiah
status: Draft
created: 2019-09-11
---

## Simple Summary

To further reduce the potential hardware acceleration of Qkchash, we propose an improved Qkchash algorithm, namely, QkchashX, which enforces left-leaning red-black (LLRB) tree implementation by incorporating the LLRB rotation statistics.

## Specification

The LLRB algorithm stores a 256-bit binary data, namely, rotation statistics.  For every rotation, the tree will
- Rotate the bits of the 256-bit binary data if the rotation is left one; or
- Rotate the bits of the 256-bit binary data and xor the data with 1 if the rotation if right one.

This essentially writes the LLRB rotation information (left = 0, right = 1) to a binary array and squeezes the array to the 256-bit binary data.

For each QkchashX computation, before the hash value is applied to Keccak (as the last step of QkchashX), the rotation statistics will be read from the LLRB tree and xor'ed with the hash value.

## Backwards Compatibility

This will be a breaking change and needs a hard fork at the end of September, a.k.a., QuarkChain Mainnet Singularity v1.2.0.

## Implications

Since the calculation requires a few additional integer shift operations, the extra cost of calculating the rotation statistics should be minimal.

## Implementation

Since we do not have native instruction to support rotating 256-bit data, we break the 256-bit data to four 64-bit integers (little-endian):

```
    std::array<uint64_t, 4> rotationStats_;
```

To rotate the 256-bit data, we use the following code:

```
    uint64_t c = 0;  // 1 for right rotation
    for (size_t i = 0; i < rotationStats_.size(); i++) {
        uint64_t nc = (rotationStats_[i] & (0x1ULL << 63)) == 0 ? 0 : 1;
        rotationStats_[i] = (rotationStats_[i] << 1) ^ c;
        c = nc;
    }
    rotationStats_[0] = rotationStats_[0] ^ c;
```

Finally, the rotation statistics will be applied to the hash value as
```
    std::array<uint64_t, 4> r_stats = tree1.getRotationStats();
    for (size_t i = 0; i < r_stats.size(); i++) {
        result[i] ^= r_stats[i];
    }
```

## Copyright
Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
