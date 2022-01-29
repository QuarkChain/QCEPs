---
qcep: 7
title: Automatic PoSW Effective Difficulty Adjustment
author: @qizhou, @junjiah
status: Draft
created: 2021-01-29
---

## Simple Summary and Motivation

To better secure QuarkChain mainnet and enable bridging assets from other chains, we propose a self-adjusting mechanism to increase the network's hash power by lowering the effective difficulty for root chain PoSW stakers.

The targeted network difficulty will match Ethereum mainnet's as of the time of writing, which is around 12.52 P (source: [2miners](https://2miners.com/eth-network-difficulty)). To compare, QuarkChain mainnet's root chain difficulty is ~91 T, roughly 1/100 of the former.

## Specification

In the configuration for full nodes, will have three more fields in `POSW_CONFIG`:

1.  `TARGETED_DIFF`: intended difficulty to match
2. `ADJUSTING_STEP`: multiplier for `DIFF_DIVIDER`, the higher the number, the lower the effective difficulty will become as the time goes. For example, with initial difficulty 1 T, current `DIFF_DIVIDER` equal to 1000 and `ADJUSTING_STEP`equal to 10:
   1. effective difficulty is 1 G, `DIFF_DIVIDER` increases 10 folds to 10k, thus network difficulty will stabilize at 10 T
   2. time goes on (until target difficulty is met), `DIFF_DIVIDER` goes to 100k, and network difficulty stabilizes at 100 T
   3. continue until `TARGETED_DIFF` is reached
3. `ADJUSTING_INTERVAL`: number of blocks between adjustments

## Rationale

Those 3 parameters will lead to smoother difficulty adjustments.

## Backwards Compatibility

Network upgrade is necessary as it's not compatible with previous versions.

## Copyright
Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
