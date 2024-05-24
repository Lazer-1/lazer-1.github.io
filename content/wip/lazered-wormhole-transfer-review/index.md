+++
title = "Lazered #1: Wormhole transfer"
date = 2024-05-02
authors = ["9oelm"]
sort_by = "date"

[extra]
katex_enable = true
+++

[Wormhole](https://docs.wormhole.com/wormhole) is a **generic message passing protocol** that enables communication between blockchains. Basically, anthing cross-chain can be created using Wormhole.

## Resources

Wormhole has an extensive list of useful resources, prominently:
- [Protocol spec](https://github.com/wormhole-foundation/wormhole/blob/main/whitepapers/)
- [Documentation](https://docs.wormhole.com/wormhole)
- [Audits](https://github.com/wormhole-foundation/wormhole-audits/)

This will be already more than enough to get us started in diving deep into the protocol.

## Transfer

This will be a comprehensive overview of how a transfer from an address on chain A to another on chain B works. Let's dive into it right away.

Let's say we have a token called MYTOKEN on Ethereum and want to transfer it to Avalanche. MYTOKEN is native on Etheruem.

