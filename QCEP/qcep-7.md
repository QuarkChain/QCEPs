---
qcep: 7
title: Automatic PoSW Effective Difficulty Adjustment
author: @qizhou, @junjiah, @pingke
status: Draft
created: 2022-03-30
---

## Simple Summary and Motivation

To better secure QuarkChain mainnet and enable bridging assets from other chains, we propose a self-adjusting mechanism to increase the network's hash power by lowering the effective difficulty for root chain PoSW stakers.

The targeted network difficulty will match Ethereum mainnet's as of the time of writing, which is around 1 PH/s (source: [2miners](https://2miners.com/eth-network-difficulty)). To compare, QuarkChain mainnet's root chain difficulty is ~1.2TH/s, roughly 1/1000 of the former.

## Specification

In the configuration for full nodes, will have four more fields in `POSW_CONFIG`:

1. `BOOST_TIMESTAMP`: timestamp to enable boost (first-time increase DIFF_DIVIDER), set BOOST_TIMESTAMP to 0 mean disable this feature;
2. `BOOST_MULTIPLIER_PER_STEP`: multiplier for `DIFF_DIVIDER` for each step (i.e. 2x);
3. `BOOST_STEPS`: number of steps to increase DIFF_DIVIDER (i.e. 10);
4. `BOOST_STEP_INTERVAL`: interval of applying next MULTIPLIER.

Therefore the new DIFF_DIVIDER equation becomes
```
if block.timestamp >= BOOST_TIMESTAMP:
   DIFF_DIVIDER *= BOOST_MULTIPLIER_PER_STEP ** min(((block.timestamp - BOOST_TIMESTAMP) // BOOST_STEP_INTERVAL) + 1, BOOST_STEPS)
```

## Rationale

Those 4 parameters will lead to smoother difficulty adjustments.

## Backwards Compatibility

Network upgrade is necessary as it's not compatible with previous versions.

## Copyright
Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
