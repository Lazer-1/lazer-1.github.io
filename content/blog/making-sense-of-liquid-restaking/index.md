+++
title = "Making sense of liquid restaking"
date = 2024-05-24T00:00:00Z
authors = ["9oelm"]
sort_by = "date"

[extra]
katex_enable = true
+++

# Evolution of staking

## The Merge

On September 15, 2022, ["The Merge" took place](https://ethereum.org/en/roadmap/merge/):

![The merge](/blog/making-sense-of-liquid-restaking/the-merge.png)

Essentially, the proof-of-stake chain, also called the Beacon Chain, running parallel to the now-obsolete proof-of-work chain was merged to the PoW chain. 

The most interesting development was that there is no more need to utilize heavy computing resources to mine blocks. 

Miners had to use a lot of electricity to solve increasingly difficult mathematical problems. Then the first miner to solve the problem wins the right to add a new block to the chain.

After The Merge, miners aren't needed anymore; instead, we call them 'validators' who help secure the network by committing their stake to validate the blocks.

Each time a new block needs to be created, a validator is randomly selected to propose a block. Then other validators attest that they have observed the block creation. Then they respectively win rewards for proposing blocks and attesting to the new blocks.

The security mechanism is simple: each validator needs to be staked with at least 32 ETH, which is about 121,600 USD as of today. If a validator produces an invalid block, their stake is 'slashed', meaning that a part of their staked ETH will be taken away. This way, validators are incentivized to only engage in an honest behavior.

## Staking

Ever since then, 'staking' has become a very crucial and popular topic in crypto. What are we going to do with it? And how can we use it to earn extra economic benefits?

The easiest way was to stake ETH to a validator so people can earn extra ETH for validating the network.

But doing that completely alone is a difficult challenge:
- One needs to have at least 32 ETH to stake.
- One needs to operate and maintain the validator, which is a program running on a computer. If not carefully configured or maintained, it might lead to slashing.

In this respect, people came up with some better idea: staking as a service and pooled staking. In a nutshell, it means that the amount of ETH that one has doesn't disqualify him from staking it, and a specialized staking service provider will run a validator node for you. In exchange, they would take a certain percentage off of the yield from the validator, but still rewarding the users sufficiently.

Some prime examples of pooled staking services are:
- [Lido](https://lido.fi/)
- [Rocket Pool](https://rocketpool.net/)
- [Swell](https://app.swellnetwork.io/stake)

## Liquid staking

The staking services soon realized that they can still keep the staked ETH in a liquid form, which could then be used in other places. It is a form of derivative because it does not represent the underlying ETH that one has. It is a representation of the staked ETH that one can withdraw if he wants to.

Then, since this staked ETH represents the underlying ETH and the ability to withdraw it, it can be regarded almost as the same thing as the underlying ETH, except that there is an additional risk involved in staking, such as slashing, smart contracts, etc.

Those who are willing to take a bit of additional risk can choose to use it in other DeFi applications. For example, Aave supports depositing or borrowing staked ETH:

![Aave stETH](/blog/making-sense-of-liquid-restaking/aave-steth.png)

This way, staked ETH is entirely kept in a liquid form, because it can be used just like its underlying asset, ETH.

## Liquid restaking and EigenLayer

<iframe width="100%" height="720" src="https://www.youtube.com/embed/01xDSwMO5U4" title="Sreeram Kannan — EigenLayr: Permissionless Feature Addition to Ethereum (ETHconomics @ Devconnect)" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>

## Ethereum and decoupling of trust and innovation

Ethereum pioneered modular blockchains. It has different layers doing different things. It decoupled trust and innovation because anyone can bring any applications on top of Ethereum network. They are 'borrowing trust' from the underlying network. The applications themselves do not need to be trusted, because the underlying network is secured by the trust network.

![Ethereum pioneered modular blockchains](/blog/making-sense-of-liquid-restaking/ethereum-pioneered-modular-blockchains.png)

### Alt-L1s fracture trust

But what if someone wants to build his own L1? This gives brith to different consensus protocols, different virtual machines, and so on. Then this again means fragmentation of trust because trust is fragmented across different blockchains.

![Fragmentation of trust](/blog/making-sense-of-liquid-restaking/fragmentation-of-trust.png)

Then we also start to see modular blockchains. Ethereum has all components (or 'modules') of consensus, which are ordering, data availability, settlement, and execution layers. But we are now seeing new blockchains being built for one of the specific modules only.

This means you can build DApps comprised of arbitrary combination of these modules.

### Modularity fractures trust

But who will be providing trust? Trust is missing here. Also, trust is fractured into smaller islands

![Further modularity](/blog/making-sense-of-liquid-restaking/further-modularity.png)

Let's say Ethereum trust network has 200bn USD staked. But if the execution layer of a DApp is based on a 10bn worth of trust only, that is the trust bottleneck and the weakest link. Therefore, the entire cost of corrupting these DApps is 10bn USD only.

### Middlewares fracture trust

Oracles, bridges, ... anything that needs a distributed validation service needs its own trust network.

This is a barrier to innovation, because if you want to build your own oracle or bridge, you would need to build your own trust network, which is extremely difficult.

![Middlewares fracture trust](/blog/making-sense-of-liquid-restaking/middlewares-fracture-trust.png)

### The problem

- Middleware: need to generate certain % of revenue from the staked assets. Massive cost.  
- DApps: need to rely on Ethereum as well as middlewares. Suffer from low and fragmented trust
- Ethereum (L1): Instead of being a decentralized trust network, its value gets split across all of the applications.

### The solution: EigenLayer

The mechanism by which the trust originating from Ethereum can be redirected into any new modules built by anyone. The core idea behind EigenLayer is restaking. You stake the liquid staking token again, subjecting yourself to an additional slashing conditions, but opting to provide new services. New distributed validation services but same validated network.

![Eigenlayer](/blog/making-sense-of-liquid-restaking/eigenlayer-0.png)

How does it improve the situation?

- Middleware: no additional cost of capital. You are restaking, so you are not taking an additional cost of capital but at the same time are taking a certain yield from the original staking in Ethereum network.
- DApps: can solely rely on core Ethereum trust instead of fractured trust.
- Ethereum: its inherent value increases. Restakers earn additional yield. Its trust network becomes more important. 

### The restaking collective 

EigenLayer is a set of smart contracts on Ethereum. EigenLayer allows stakers to opt in for greater yield while taking a greater risk.

When a user stakes, he needs to set the withdrawal address as EigenLayer smart contract instead of a personal wallet address because if something goes wrong in the service that is provided with his restaked ETH, he might lose a part of his stake.

Instead of securing different networks with a different trust mechanisms, now Ethereum steps in to secure it all. And that's only one trust network to rule them all, which is convenient and easy.

![Unified trust](/blog/making-sense-of-liquid-restaking/unified-trust.png)

Now the cost of capital used to secure Ethereum network only, can bring amortization of cost of capital, if it is used across multiple networks.

> EigenLayer wants to do the equivalent of rehypothecation in traditional finance: re-using previously pledged collateral as the collateral for a new loan. [^1]

### EigenLayer advantages

- Increased security for protocols: protocols (or AVS, actively validated services) are now sharing Ethereum's immense security without having to launch their own trust networks.
- Protocol governance: there are serious tradeoffs in protocol governance directions. Agility is fast, but democracy is slow. But we can combine these two. Ethereum is stable, and EigenLayer allows protocols to control the amount of pooled security needed and validators to control the amount they supply. It is a competitive economic market of security. For example, one AVS might say, it needs only 1 stETH to validate what it is doing because what it is doing is trivial and is worth only of 1 stETH's security. Another AVS might say, it needs 100 stETH to validate what it is doing because it is a critical service. This is a democracy, but it is also agile.

![agility-vs-democracy.png](/blog/making-sense-of-liquid-restaking/agility-vs-democracy.png)

- Improved capital efficiency: no need to create a new token, trust network, bootstrap custom validators, etc. The capital can be used somewhere else.

<!-- ### Vitalik's thoughts: EigenLayer's risk -->

# Swell

Swell is a non-custodial liquid staking/restaking protocol built on Ethereum. It used to only support ETH staking, but from January 2024, it began to support native restaking to EigenLayer.

Restaking unlocks new opportunities for stakers: they no longer have to lock their liquidity in EigenLayer by restaking their liquid staking token like swETH. Instead, Swell mints rswETH (restaked Swell Eth) when you restake your swETH.

Then, rswETH can just be used like ETH, transferred around or deposited into another DeFi protocol.

<!-- ## Swell L2

Swell is planning to launch its own L2, and many think that it is the epitome of the state-of-the-art modular blockchain.

![swell-l2-architecture.png](/blog/making-sense-of-liquid-restaking/swell-l2-architecture.png)

Each specific blockchain module will handle a specific layer, as described in the image above. -->

<!-- # The future of restaking landscape -->

# References

[^1]: [[Bankless] Vitalik’s Warning and EigenLayer's Promise](https://www.bankless.com/vitaliks-warning-and-eigenlayers-promise) 

- [[Coinbase] Restaking: Everything Old is New Again](https://www.coinbase.com/institutional/research-insights/research/market-intelligence/restaking-everything-old-is-new-again)
- [[Coinbase] Understanding the EigenLayer AVS Landscape](https://www.coinbase.com/blog/eigenlayer)
- [[Consensys] EigenLayer: Decentralized Ethereum Restaking Protocol Explained](https://consensys.io/blog/eigenlayer-a-restaking-primitive)
- [[Ethereum] The Merge](https://ethereum.org/en/roadmap/merge/)
- [[Four pillars] Swell, L2 for the Optimal Restaking Experience](https://4pillars.io/en/articles/swell-l2-for-the-optimal-restaking-experience/public)
- [[EigenLayer Forum] Learn about EigenLayer](https://forum.eigenlayer.xyz/t/learn-about-eigenlayer/3418)
- [[Medium] Liquid Restaking Wars: the Next Lido](https://medium.com/oregon-blockchain-group/liquid-restaking-wars-the-next-lido-5e8fd152d799)
- [[Renzo] Renzo FAQs](https://docs.renzoprotocol.com/docs/product/renzo-faqs)
- [[Substack] Eigenlayer | What Is It?](https://mazii08.substack.com/p/eigenlayer-what-is-it)
- [[Swell Blog] Swell L2 Pre-Launch Deposits | Space Recap](https://www.swellnetwork.io/post/l2-pre-launch-space-recap)
- [[Swell Blog] What is Restaking?](https://www.swellnetwork.io/post/what-is-restaking)
- [[Swell Blog] AVS Selection Framework](https://www.swellnetwork.io/post/avs-selection-framework)
- [[Swell Blog] Swell Collaborates with AltLayer, EigenDA, and Chainlink to Launch L2 for Restaking with Polygon CDK](https://www.swellnetwork.io/post/restaking-l2)
- [[Swell Blog] 3 Things You Need to Know About rswETH v2](https://www.swellnetwork.io/post/3-things-you-need-to-know-about-rsweth-v2)
- [[Twitter] Abi's description of Swell L2](https://unrollnow.com/status/1789672050321408366)
- [[Twitter] Swell Network; a Deep Dive into the Most Interesting L2 by Kairos Research](https://twitter.com/Kairos_Res/status/1788597081373835667)
- [[Youtube] Sreeram Kannan — EigenLayr: Permissionless Feature Addition to Ethereum (ETHconomics @ Devconnect)](https://www.youtube.com/watch?v=01xDSwMO5U4)
- [[Youtube] Bankless: EigenLayer with Sreeram Kannan](https://youtu.be/AhxVcAvAhQ0)
- [[Vitalik] Don't overload Ethereum's consensus](https://vitalik.eth.limo/general/2023/05/21/dont_overload.html)
