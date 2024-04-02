+++
title = "Making sense of Starknet's L1↔L2 Bridge"
slug = "making-sense-of-starknets-l1-l2-starknet-bridge"
date = "2024-03-28"
authors = ["9oelm"]
+++

With the recent STRK provisions program, Starknet has started gain public attention, and people want to know more about it. But still on the technical side, there is a lot that is missing. So this writing intends to give a deep understanding of how the L1↔L2 bridge work on Starknet.

Starknet is a Layer 2 ZK Rollup for Ethereum. Communication with L1 still needs to happen. 

# Overview of L1-L2 messaging

Here's an overview of L1-L2 messaging on Starknet. Understanding of such a concept is crucial to make sense of how token bridges work. Here's [a diagram from Starknet's official documentation](https://docs.starknet.io/documentation/architecture_and_concepts/Network_Architecture/messaging-mechanism/):

![L2→L1 message mechanism](/blog/making-sense-of-starknets-l1-l2-starknet-bridge/l2_l1_messaging_mechanism.png)

# `LegacyBridge`

Starknet used to have a bridge per token. This means that for every single token that was meant to be bridged, the same bridge has to be deployed again. This bridge is still live in production and still serves most of the bridge tokens to this date. The list of tokens that make use of `LegacyBridge` can be discovered at [https://github.com/starknet-io/starknet-addresses/blob/master/bridged_tokens/mainnet.json](https://github.com/starknet-io/starknet-addresses/blob/master/bridged_tokens/mainnet.json).

Some of them are:

![Bridged assets](/blog/making-sense-of-starknets-l1-l2-starknet-bridge/bridged_tokens.png)

You can see that each token's bridge address is different from one another, which means that `LegacyBridge` is only limited to one deployment per token.