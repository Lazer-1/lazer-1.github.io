+++
title = "How Jettons work on TON with sharding in mind"
date = 2024-07-01
authors = ["9oelm"]
sort_by = "date"

[extra]
katex_enable = true
+++

## Sharding in TON

TON employs an unique approach to build the "blockchain of blockchains" by ['sharding' its chains](https://docs.ton.org/develop/blockchain/sharding-lifecycle). The original concept of sharding comes from database design. It is a way to split a big database into smaller, manageable pieces.

For example: imagine you have a huge book of contacts. It's so big that it's hard to find anything quickly. Sharding is like dividing that book into smaller address books based on last names:
- Book A-G
- Book H-M
- Book N-S
- Book T-Z

Now, when you need to find someone, you only have to look in one smaller book instead of searching through the entire giant book. This makes finding information faster and easier.

In database terms:
- Each smaller book is a "shard"
- The data is spread across multiple servers instead of one big server
- This helps the database handle more information and work faster

When this concept is applied to blockchain, it enables multiple pieces of data to be processed in parallel. This makes blockchain significantly faster and more scalable.

## How sharding affects smart contract design on TON

But there's a gotcha with sharding on TON:
- [A smart contract is only allowed synchronous access to its own local state](https://blog.ton.org/six-unique-aspects-of-ton-blockchain-that-will-surprise-solidity-developers#3-your-smart-contract). Smart contracts cannot access other contracts' state because accessing the state cannot be synchronous and atomic.
- [Calls between smart contracts are always asynchronous.](https://blog.ton.org/six-unique-aspects-of-ton-blockchain-that-will-surprise-solidity-developers#2-calls-between-smart)
- [Unbounded data structure needs to be designed as sharded contracts](https://blog.ton.org/six-unique-aspects-of-ton-blockchain-that-will-surprise-solidity-developers#5-you-should-not). This means if you had `mapping(...)` on Ethereum, you should expand this into contracts. For example, if the `mapping(address => uint256)` manages token balances of users, there would be as many contracts as users on TON, each representing a token balance belonging to each user. This way, multiple contracts can be processed in parallel

This is quite a huge change for any developers coming from EVM chains, because they don't have to worry about access to other contracts' state or having to deal with an asynchronous operation because there cannot be one on EVM.

Now, we will review this concept with a practical example: Jettons.

## Smart contract design of Jettons

Jettons are the token standard on TON, just like ERC20 on Ethereum. 
- The code is open sourced at [`ton-blockchain/token-contract`](https://github.com/ton-blockchain/token-contract/tree/21e7844fa6dbed34e0f4c70eb5f0824409640a30/ft). 
- [The standard is called TEP-74, viewable at `ton-blockchain/TEPs`](https://github.com/ton-blockchain/TEPs/blob/master/text/0074-jettons-standard.md). 
- [Another relevant standard is the 'content' standard, TEP-64](https://github.com/ton-blockchain/TEPs/blob/master/text/0064-token-data-standard.md#jetton-metadata-attributes). This dictates the structure of the `cell content` stored in [`jetton-minter.fc`](https://github.com/ton-blockchain/token-contract/blob/21e7844fa6dbed34e0f4c70eb5f0824409640a30/ft/jetton-minter.fc). 

The important contracts to look at are:
- [`jetton-minter.fc`](https://github.com/ton-blockchain/token-contract/blob/21e7844fa6dbed34e0f4c70eb5f0824409640a30/ft/jetton-minter.fc)
- [`jetton-wallet.fc`](https://github.com/ton-blockchain/token-contract/blob/21e7844fa6dbed34e0f4c70eb5f0824409640a30/ft/jetton-wallet.fc)

[`jetton-minter.fc`](https://github.com/ton-blockchain/token-contract/blob/21e7844fa6dbed34e0f4c70eb5f0824409640a30/ft/jetton-minter.fc) is the 'parent' contract, that contains global information about the token, like total supply, name, symbol, and admin address.

The data stored in `jetton-minter.fc` are the following:

<iframe frameborder="0" scrolling="no" style="width:100%; height:268px;" allow="clipboard-write" src="https://emgithub.com/iframe.html?target=https%3A%2F%2Fgithub.com%2Fton-blockchain%2Ftoken-contract%2Fblob%2F21e7844fa6dbed34e0f4c70eb5f0824409640a30%2Fft%2Fjetton-minter.fc%23L9-L17&style=atom-one-dark&type=code&showBorder=on&showLineNumbers=on&showFileMeta=on&showFullPath=on&showCopy=on"></iframe>

`load_data()` returns 4 things: `int total_supply, slice admin_address, cell content, cell jetton_wallet_code`. 
- `total_supply` is the amount of total tokens minted. 
- `admin_address` is the address of the admin who can control the minter contract.
- `content` is the cell of data that abides by [TEP-64](https://github.com/ton-blockchain/TEPs/blob/master/text/0064-token-data-standard.md#jetton-metadata-attributes).

`get_data()` returns the persistent contract storage cell. Remember everything on TON is stored in a cell. Then, `begin_parse` converts `cell` into `slice`. The reason is that all `load_*` methods only work on `slice` type, not `cell`. The data bits and references to other cells from the cell can be obtained by loading them from the `slice`. 

In other words, a `slice` is a contiguous “sub-cell” of an existing cell, containing some of its bits of data and some of its references. Essentially, a slice is a read-only view for a subcell of a cell. Slices are used for unpacking data previously stored (or serialized) in a cell or a tree of cells.

`load_coins()` returns `total_supply` of type `VarUInteger16 = Coins`. The expected serialization of $x$ consists of a 4-bit unsigned big-endian integer $l$ (denoting the length of the following value $x$), followed by an $8 \times l$-bit unsigned big-endian representation of $x$. The serialization is only 124 bits long (not 128!). The maximum amount of `VarUInteger16` is $2^{120} - 1$. Anyway, when it comes to manipulating the amount of Toncoin or Jettons, we always use `load_coins()` instead of something like `load_uint()`.

`load_msg_addr` loads `MsgAddress`. This function is used whenever you need to load a TON address stored in a cell.

`load_ref()` loads a reference to another cell. In this case, loading cells containing `content` and `jetton_wallet_code`.

Now, let's look at how the data is saved, which is just the opposite of `load_data()`:

<iframe frameborder="0" scrolling="no" style="width:100%; height:268px;" allow="clipboard-write" src="https://emgithub.com/iframe.html?target=https%3A%2F%2Fgithub.com%2Fton-blockchain%2Ftoken-contract%2Fblob%2F21e7844fa6dbed34e0f4c70eb5f0824409640a30%2Fft%2Fjetton-minter.fc%23L19-L27&style=atom-one-dark&type=code&showBorder=on&showLineNumbers=on&showFileMeta=on&showFullPath=on&showCopy=on"></iframe>

`set_data` just takes a `cell` and sets it as persistent contract data.

`begin_cell()` is something special; it creates a new `builder`. Data bits and references to other cells can be stored in a `builder`, and then the `builder` can be finalized to a new cell by calling `end_cell()`. The reason we are using `end_cell()` is that the only possible parameter of `set_data` is of type `cell`, which is stored permanently by the contract.

If you want to know how you can serialize `content` into a cell, check out the code:

<iframe frameborder="0" scrolling="no" style="width:100%; height:100px;" allow="clipboard-write" src="https://emgithub.com/iframe.html?target=https%3A%2F%2Fgithub.com%2Fton-blockchain%2Fminter-contract%2Fblob%2F074b7d5f45f43552146fdf54f972020b2757bc18%2Fbuild%2Fjetton-minter.deploy.ts%23L127&style=atom-one-dark&type=code&showBorder=on&showLineNumbers=on&showFileMeta=on&showFullPath=on&showCopy=on"></iframe>

<iframe frameborder="0" scrolling="no" style="width:100%; height:751px;" allow="clipboard-write" src="https://emgithub.com/iframe.html?target=https%3A%2F%2Fgithub.com%2Fton-blockchain%2Fminter-contract%2Fblob%2F074b7d5f45f43552146fdf54f972020b2757bc18%2Fbuild%2Fjetton-minter.deploy.ts%23L42-L73&style=atom-one-dark&type=code&showBorder=on&showLineNumbers=on&showFileMeta=on&showFullPath=on&showCopy=on"></iframe>

`buildTokenMetadataCell` is exactly how it is done. But because this is not the main goal of this post, we're not gonna explain this in detail.

Now, let us look at `recv_internal` of `jetton-minter.fc`, which is a function that is invoked when an [internal message](https://docs.ton.org/develop/smart-contracts/guidelines/internal-messages) arrives at this contract.

<iframe frameborder="0" scrolling="no" style="width:100%; height:1465px;" allow="clipboard-write" src="https://emgithub.com/iframe.html?target=https%3A%2F%2Fgithub.com%2Fton-blockchain%2Ftoken-contract%2Fblob%2F21e7844fa6dbed34e0f4c70eb5f0824409640a30%2Fft%2Fjetton-minter.fc%23L42-L107&style=atom-one-dark&type=code&showBorder=on&showLineNumbers=on&showFileMeta=on&showFullPath=on&showCopy=on"></iframe>

Note that all of these function signatures are correct:

```js
() recv_internal(int balance, int msg_value, cell in_msg_full, slice in_msg_body) {}
() recv_internal(int msg_value, cell in_msg_full, slice in_msg_body) {}
() recv_internal(cell in_msg_full, slice in_msg_body) {}
() recv_internal(slice in_msg_body) {}
```

But you can just use whichever function signature that best suits your purpose and gas fee management.

The function parameters are as follows (check [4.4.5 of TON Blockchain docs](https://test.ton.org/tblkch.pdf)):
- `int balance`: The current balance of TON of the smart contract (after crediting the value of the inbound message) in nanograms.
- `int msg_value`: the amount of TON sent in the message in nanograms.
- `cell in_msg_full`: the inbound message passed as cell, containing the full message, including the message body.
- `slice in_msg_body`: the 'body' of the inbound message.

If you don't need some of the parameters, you can use the function signature with less parameters.

The first line of the function starts with [`slice_empty?()`](https://docs.ton.org/develop/func/cookbook#how-to-determine-if-slice-is-empty).

This returns true if there is at least one bit of data or one ref (if you want to check if slice only has data and NOT refs, use `slice_data_empty`).

Because we don't want to process empty message body, we immediately return.

Next, we process the flags. There isn't a lot of information about the "bounced" flag, but by looking at the code we can assume that the bounced flag is a single bit the end of the 32-bytes long flags. If it is bounced, it will be 1. If not, it will be 0. Like: 0b01010101011111....._1_ (or 0).

Next, the sender address is loaded by `sender_address`, `op`, `query_id` are loaded.

Note that `sender_address` is from `in_msg_full`, while `op` and `query_id` are from `in_msg_body`, marking the beginning of the message body.

[The message body's structure](https://docs.ton.org/develop/smart-contracts/guidelines/internal-messages#internal-message-body) is always 32-bit (big-endian) unsigned integer `op`, followed by 64-bit (big-endian) unsigned integer `query_id`. Then the rest of the message body depends on the `op`.

We've already covered `load_data()`, so we pass on this one.

Now, we are finally dealing with different `op`s. Each if statement handles one `op`.

We need to look at [op-codes.fc](https://github.com/ton-blockchain/token-contract/blob/21e7844fa6dbed34e0f4c70eb5f0824409640a30/ft/op-codes.fc) first:

<iframe frameborder="0" scrolling="no" style="width:100%; height:268px;" allow="clipboard-write" src="https://emgithub.com/iframe.html?target=https%3A%2F%2Fgithub.com%2Fton-blockchain%2Ftoken-contract%2Fblob%2F21e7844fa6dbed34e0f4c70eb5f0824409640a30%2Fft%2Fop-codes.fc%23L1-L9&style=atom-one-dark&type=code&showBorder=on&showLineNumbers=on&showFileMeta=on&showFullPath=on&showCopy=on"></iframe>

`op-codes.fc` is included into `jetton-minter.fc` upon build, making `op::*` functions callable in it:

<iframe frameborder="0" scrolling="no" style="width:100%; height:100px;" allow="clipboard-write" src="https://emgithub.com/iframe.html?target=https%3A%2F%2Fgithub.com%2Fton-blockchain%2Ftoken-contract%2Fblob%2F21e7844fa6dbed34e0f4c70eb5f0824409640a30%2Fft%2Fcompile.sh%23L3-L3&style=atom-one-dark&type=code&showBorder=on&showLineNumbers=on&showFileMeta=on&showFullPath=on&showCopy=on"></iframe>

An opcode is nothing but a number. For example, `int op::transfer() asm "0xf8a7ea5 PUSHINT";` means transfer has an opcode of `0xf8a7ea5`.

Do note that it is also possible to define an opcode by using a newer syntax:

```ts
const int op::transfer = 0xf8a7ea5;
```

Let's go back to `if (op == op::mint())` and have a look at what's inside the if statement:

```js
if (op == op::mint()) {
    throw_unless(73, equal_slices(sender_address, admin_address));
    slice to_address = in_msg_body~load_msg_addr();
    int amount = in_msg_body~load_coins();
    cell master_msg = in_msg_body~load_ref();
    slice master_msg_cs = master_msg.begin_parse();
    master_msg_cs~skip_bits(32 + 64); ;; op + query_id
    int jetton_amount = master_msg_cs~load_coins();
    mint_tokens(to_address, jetton_wallet_code, amount, master_msg);
    save_data(total_supply + jetton_amount, admin_address, content, jetton_wallet_code);
    return ();
}
```

`throw_unless(73, equal_slices(sender_address, admin_address));` will throw if `sender_address` is not equal to `admin_address`. The reason is pretty obvious; if new tokens can be minted by anyone, that is an immediate vulnerability. The error code is `73`.

The labels of the error codes are located at `JettonConstants.ts`:

<iframe frameborder="0" scrolling="no" style="width:100%; height:394px;" allow="clipboard-write" src="https://emgithub.com/iframe.html?target=https%3A%2F%2Fgithub.com%2Fton-blockchain%2Ftoken-contract%2Fblob%2F21e7844fa6dbed34e0f4c70eb5f0824409640a30%2Fwrappers%2FJettonConstants.ts%23L16-L28&style=atom-one-dark&type=code&showBorder=on&showLineNumbers=on&showFileMeta=on&showFullPath=on&showCopy=on"></iframe>

From this, we know that the error code `73` means "not admin".

- [[TON Blog] How to shard your TON smart contract and why - studying the anatomy of TON's Jettons](https://blog.ton.org/how-to-shard-your-ton-smart-contract-and-why-studying-the-anatomy-of-tons-jettons)
- [[TON Blog] Six unique aspects of TON Blockchain that will surprise Solidity developers](https://blog.ton.org/six-unique-aspects-of-ton-blockchain-that-will-surprise-solidity-developers)
- [[Excalidraw] Contracts design diagram](https://excalidraw.com/#json=G8O---P5YSw45M_Fv9rzW,zU2SGCurOzwfre59EImQtQ)
- [[Github] awesome-ton-smart-contracts](https://github.com/dkeysil/awesome-ton-smart-contracts)
- [[Youtube] Technical Demo: Sharded Smart Contract Architecture for Smart Contract Developers](https://youtu.be/svOadLWwYaM)
- [TVM Whitepaper](https://docs.everscale.network/tvm.pdf)