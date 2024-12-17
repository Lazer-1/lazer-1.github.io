+++
title = "How Jettons work on TON with sharding in mind (Part 1)"
date = 2024-07-01
authors = ["9oelm"]
sort_by = "date"

[extra]
katex_enable = true
+++

# Sharding in TON

TON employs a unique approach to build the "blockchain of blockchains" by ['sharding' its chains](https://docs.ton.org/develop/blockchain/sharding-lifecycle). The original concept of sharding comes from database design. It is a way to split a big database into smaller, manageable pieces.

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

# How sharding affects smart contract design on TON

But there's a gotcha with sharding on TON:
- [A smart contract is only allowed synchronous access to its own local state](https://blog.ton.org/six-unique-aspects-of-ton-blockchain-that-will-surprise-solidity-developers#3-your-smart-contract). Smart contracts cannot access other contracts' state because accessing the state cannot be synchronous and atomic.
- [Calls between smart contracts are always asynchronous.](https://blog.ton.org/six-unique-aspects-of-ton-blockchain-that-will-surprise-solidity-developers#2-calls-between-smart)
- [Unbounded data structure needs to be designed as sharded contracts](https://blog.ton.org/six-unique-aspects-of-ton-blockchain-that-will-surprise-solidity-developers#5-you-should-not). This means if you had `mapping(...)` on Ethereum, you should expand this into contracts. For example, if the `mapping(address => uint256)` manages token balances of users, there would be as many contracts as users on TON, each representing a token balance belonging to each user. This way, multiple contracts can be processed in parallel

This is quite a huge change for any developers coming from EVM chains, because they don't have to worry about access to other contracts' state or having to deal with an asynchronous operation because there cannot be one on EVM.

Now, we will review this concept with a practical example: Jettons.

# Smart contract design of Jettons

Jettons are the token standard on TON, just like ERC20 on Ethereum. 
- The code is open sourced at [`ton-blockchain/token-contract`](https://github.com/ton-blockchain/token-contract/tree/21e7844fa6dbed34e0f4c70eb5f0824409640a30/ft). 
- [The standard is called TEP-74, viewable at `ton-blockchain/TEPs`](https://github.com/ton-blockchain/TEPs/blob/master/text/0074-jettons-standard.md). 
- [Another relevant standard is the 'content' standard, TEP-64](https://github.com/ton-blockchain/TEPs/blob/master/text/0064-token-data-standard.md#jetton-metadata-attributes). This dictates the structure of the `cell content` stored in [`jetton-minter.fc`](https://github.com/ton-blockchain/token-contract/blob/21e7844fa6dbed34e0f4c70eb5f0824409640a30/ft/jetton-minter.fc). 

The important contracts to look at are:
- [`jetton-minter.fc`](https://github.com/ton-blockchain/token-contract/blob/21e7844fa6dbed34e0f4c70eb5f0824409640a30/ft/jetton-minter.fc)
- [`jetton-wallet.fc`](https://github.com/ton-blockchain/token-contract/blob/21e7844fa6dbed34e0f4c70eb5f0824409640a30/ft/jetton-wallet.fc)

## jetton-minter

[`jetton-minter.fc`](https://github.com/ton-blockchain/token-contract/blob/21e7844fa6dbed34e0f4c70eb5f0824409640a30/ft/jetton-minter.fc) is the 'parent' contract, that contains global information about the token, like total supply, name, symbol, and admin address.

The data stored in `jetton-minter.fc` are the following:

<iframe frameborder="0" scrolling="no" style="width:100%; height:268px;" allow="clipboard-write" src="https://emgithub.com/iframe.html?target=https%3A%2F%2Fgithub.com%2Fton-blockchain%2Ftoken-contract%2Fblob%2F21e7844fa6dbed34e0f4c70eb5f0824409640a30%2Fft%2Fjetton-minter.fc%23L9-L17&style=atom-one-dark&type=code&showBorder=on&showLineNumbers=on&showFileMeta=on&showFullPath=on&showCopy=on"></iframe>

`load_data()` returns 4 things: `int total_supply, slice admin_address, cell content, cell jetton_wallet_code`. 
- `total_supply` is the amount of total tokens minted. 
- `admin_address` is the address of the admin who can control the minter contract.
- `content` is the cell of data that abides by [TEP-64](https://github.com/ton-blockchain/TEPs/blob/master/text/0064-token-data-standard.md#jetton-metadata-attributes).
- `jetton_wallet_code` is [the actual code of `jetton-wallet.fc` contract](https://github.com/ton-blockchain/token-contract/blob/21e7844fa6dbed34e0f4c70eb5f0824409640a30/sandbox_tests/JettonWallet.spec.ts#L36). If someone who doesn't have a wallet yet receives a coin, he will receive the message with `jetton-wallet` code so his wallet can be deployed.

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

This returns false if there is at least one bit of data or one ref (if you want to check if slice only has data and NOT refs, use `slice_data_empty`).

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

An opcode is nothing but a number. For example, `int op::transfer() asm "0xf8a7ea5 PUSHINT";` means transfer has an opcode of `0xf8a7ea5`. We will cover how this opcode is derived in the later section.

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

Then, we extract necessary variables from `in_msg_body`: `to_address`, `amount`, `master_msg`. `master_msg` is another cell referenced from `in_msg_body`, so we `begin_parse()` it. Then, `master_msg_cs` acts as another message body, so we skip `op` and `query_id`. `jetton_amount` is loaded from `master_msg_cs`.

One oddity we could find is that `amount` needs to equal `jetton_amount`. 
<!-- not sure why yet -->

Then, `mint_tokens` is called. This sends a message to `to_address` to mint a token of `amount`. We will look at how `jetton-wallet.fc` behaves when receiving this message later.

Let's look at how `mint_tokens` is used:

<iframe frameborder="0" scrolling="no" style="width:100%; height:331px;" allow="clipboard-write" src="https://emgithub.com/iframe.html?target=https%3A%2F%2Fgithub.com%2Fton-blockchain%2Ftoken-contract%2Fblob%2F21e7844fa6dbed34e0f4c70eb5f0824409640a30%2Fft%2Fjetton-minter.fc%23L29-L40&style=atom-one-dark&type=code&showBorder=on&showLineNumbers=on&showFileMeta=on&showFullPath=on&showCopy=on"></iframe>

`.store_uint(0x18, 6)` stores 0x18 = 0b011000 first.
- First bit is 0, which is 1 bit prefix which indicates that it is `int_msg_info`.
- The next `110` means 
  - 1: Instant Hypercube Routing is disabled, 
  - 1: messages can be bounced, 
  - 0: message is not the result of bouncing itself.

Then there should be sender address, however since it anyway will be rewritten with the same effect any valid address may be stored there. The shortest valid address serialization is that of `addr_none` and it serializes as a two-bit string 00.

Then, `total_supply` is updated to `total_supply + jetton_amount` as expected, and we return an empty tuple from the function.

After the summation, `.store_uint(4 + 2 + 1, 1 + 4 + 4 + 64 + 32 + 1 + 1 + 1)` is the same as `.store_uint(7, 108)`.

The meaning of `1 + 4 + 4 + 64 + 32 + 1 + 1 + 1` is the following [^1]:

> First bit stands for empty extra-currencies dictionary.

> Then we have two 4-bit long fields. They encode 0 as `VarUInteger 16`. In fact, since `ihr_fee` and `fwd_fee` will be overwritten, we may as well put there zeroes.

> Then we put zero to `created_lt` and `created_at` fields. Those fields will be overwritten as well; however, in contrast to fees, these fields have a fixed length and are thus encoded as 64- and 32-bit long strings. (we had already serialized the message header and passed to init/body at that moment)

> Next zero-bit means that there is no init field.

> The last zero-bit means that `msg_body` will be serialized in-place.

> After that, message body (with arbitrary layout) is encoded.

Now, you must be wondering where this particular order of serialization came from. This rule in TON is called TL-B scheme. Let us look into that closely before going any further.

### TL-B schemes

TL-B stands for Type Language - Binary. It is a language designed to describe the type system, constructors and functions. Even the `message` that we send can be described by TL-B because it has a certain structure: 

<iframe frameborder="0" scrolling="no" style="width:100%; height:226px;" allow="clipboard-write" src="https://emgithub.com/iframe.html?target=https%3A%2F%2Fgithub.com%2Fton-blockchain%2Fton%2Fblob%2F5c392e0f2d946877bb79a09ed35068f7b0bd333a%2Fcrypto%2Fblock%2Fblock.tlb%23L155-L161&style=atom-one-dark&type=code&showBorder=on&showLineNumbers=on&showFileMeta=on&showFullPath=on&showCopy=on"></iframe>

This is `MessageRelaxed` type that we send as a parameter of `send_raw_message`. We note that the message has three parts:
1. `info`
2. `init`
3. `body`

We will only deal with `MessageRelaxed` instead of `Message` for the purpose of explanation:

<iframe frameborder="0" scrolling="no" style="width:100%; height:142px;" allow="clipboard-write" src="https://emgithub.com/iframe.html?target=https%3A%2F%2Fgithub.com%2Fton-blockchain%2Fton%2Fblob%2F5c392e0f2d946877bb79a09ed35068f7b0bd333a%2Fcrypto%2Fblock%2Fblock.tlb%23L159-L161&style=atom-one-dark&type=code&showBorder=on&showLineNumbers=on&showFileMeta=on&showFullPath=on&showCopy=on"></iframe>

`message$_` is the constructor. Constructor tag is the postfix after the dollar sign: `$_`. In this case, `_` means there is no prefix of any bits at the beginning of the structure.

Also, note that the type declaration is different from some languages like Typescript, where the name of the type is on LHS, like `type MyType = number`. In TL-B, that is reverse. 

It refers to three different custom types:
1. `info:CommonMsgInfoRelaxed`

    <iframe frameborder="0" scrolling="no" style="width:100%; height:205px;" allow="clipboard-write" src="https://emgithub.com/iframe.html?target=https%3A%2F%2Fgithub.com%2Fton-blockchain%2Fton%2Fblob%2F5c392e0f2d946877bb79a09ed35068f7b0bd333a%2Fcrypto%2Fblock%2Fblock.tlb%23L135-L140&style=atom-one-dark&type=code&showBorder=on&showLineNumbers=on&showFileMeta=on&showFullPath=on&showCopy=on"></iframe>

    Either of the definition can be used. We can know that the message falls into either type when deserializing, by looking at the prefix. 
    
    `int_msg_info$0` starts with a 1-bit-long prefix of `0`. Similarly, `ext_out_msg_info$11` starts with 2-bits-long prefix of `11`.

    Each `Bool` type accounts for a single bit, being either `0` or `1`:

    <iframe frameborder="0" scrolling="no" style="width:100%; height:121px;" allow="clipboard-write" src="https://emgithub.com/iframe.html?target=https%3A%2F%2Fgithub.com%2Fton-blockchain%2Fton%2Fblob%2F5c392e0f2d946877bb79a09ed35068f7b0bd333a%2Fcrypto%2Fblock%2Fblock.tlb%23L4-L5&style=atom-one-dark&type=code&showBorder=on&showLineNumbers=on&showFileMeta=on&showFullPath=on&showCopy=on"></iframe>

    `MsgAddress` and `MsgAddressInt` is defined as the following:

    <iframe frameborder="0" scrolling="no" style="width:100%; height:310px;" allow="clipboard-write" src="https://emgithub.com/iframe.html?target=https%3A%2F%2Fgithub.com%2Fton-blockchain%2Fton%2Fblob%2F5c392e0f2d946877bb79a09ed35068f7b0bd333a%2Fcrypto%2Fblock%2Fblock.tlb%23L100-L110&style=atom-one-dark&type=code&showBorder=on&showLineNumbers=on&showFileMeta=on&showFullPath=on&showCopy=on"></iframe>

    No need to digest everything about the address. For now, we just understand that this can be an address.

    Next is `CurrencyCollection`:

    <iframe frameborder="0" scrolling="no" style="width:100%; height:163px;" allow="clipboard-write" src="https://emgithub.com/iframe.html?target=https%3A%2F%2Fgithub.com%2Fton-blockchain%2Fton%2Fblob%2F5c392e0f2d946877bb79a09ed35068f7b0bd333a%2Fcrypto%2Fblock%2Fblock.tlb%23L121-L124&style=atom-one-dark&type=code&showBorder=on&showLineNumbers=on&showFileMeta=on&showFullPath=on&showCopy=on"></iframe>

    We don't care about `ExtraCurrencyCollection` for now because it's not used. `Grams` is defined as below:

    <iframe frameborder="0" scrolling="no" style="width:100%; height:100px;" allow="clipboard-write" src="https://emgithub.com/iframe.html?target=https%3A%2F%2Fgithub.com%2Fton-blockchain%2Fton%2Fblob%2F5c392e0f2d946877bb79a09ed35068f7b0bd333a%2Fcrypto%2Fblock%2Fblock.tlb%23L116&style=atom-one-dark&type=code&showBorder=on&showLineNumbers=on&showFileMeta=on&showFullPath=on&showCopy=on"></iframe>

    `VarUInteger` is simply `var_uint$_ {n:#} len:(#< n) value:(uint (len * 8)) = VarUInteger n;`. So `VarUInteger n` is just another notation for saying "An unsigned integer that is `n` bytes (not bits) long`.

    From [the official paper detailing TON](https://docs.ton.org/tblkch.pdf):

    > If one wants to represent $x$ nanograms, one selects an integer $l < 16$ such that $x < 2^{8l}$, and serializes first $l$ as an unsigned 4-bit integer, then $x$ itself as an unsigned $8l$-bit integer. 
    
    > Notice that four zero bits represent a zero amount of $Grams$. Recall that the original total supply of $Grams$ is fixed at five billion (i.e., $5 · 1018 < 2^{63}$ nanograms), and is expected to grow very slowly. Therefore, all the amounts of $Grams$ encountered in practice will fit in unsigned or even signed 64-bit integers. The validators may use the 64-bit integer representation of Grams in their internal computations; however, the serialization of these values the blockchain is another matter.

    After that, `ihr_fee` and `fwd_fee` are also of type `Grams`, so we know what to do for them too. `created_lt` and `created_at` are simply `uint64` and `uint32` fields.

2. `init:(Maybe (Either StateInit ^StateInit))`

    So what are `Maybe` and `Either`?

    <iframe frameborder="0" scrolling="no" style="width:100%; height:163px;" allow="clipboard-write" src="https://emgithub.com/iframe.html?target=https%3A%2F%2Fgithub.com%2Fton-blockchain%2Fton%2Fblob%2F5c392e0f2d946877bb79a09ed35068f7b0bd333a%2Fcrypto%2Fblock%2Fblock.tlb%23L8-L11&style=atom-one-dark&type=code&showBorder=on&showLineNumbers=on&showFileMeta=on&showFullPath=on&showCopy=on"></iframe>

    According to TL-B definition, when a type is `Maybe X`, it's prefixed with `0` or `1`. When it's prefixed with `0`, nothing is there for you to deserialize; it's an empty piece of data. When `1`, it contains a value of type `X`. For example, `(Maybe int32)` can be just `0` or `100000000000000000000000000000011` (in binary) to denote 3 in decimal, where the leftmost bit is a prefix that tells that there is data.

    `Either` works in a similar way; if prefixed with `0`, that means the data will contain `X`. If `1`, the data will contain `Y`. For example, possible values of type `(Either Bool int32)` are: `00` (`Bool` and `false`), `01` (`Bool` and `true`), or something like `100000000000000000000000000000011` (`int32` and 3 in decimal).

    Back to the original type, we have `(Either StateInit ^StateInit)`. `^` means the field is a reference to another cell of the same type, instead of being an explicit field in the current cell.

    But before `Either`, we have `(Maybe (Either StateInit ^StateInit))`, so this means that we might or might not have `StateInit`, and if we have it, it is either stored in the current cell, or a reference to another cell.

    `StateInit` is defined as follows:

    <iframe frameborder="0" scrolling="no" style="width:100%; height:142px;" allow="clipboard-write" src="https://emgithub.com/iframe.html?target=https%3A%2F%2Fgithub.com%2Fton-blockchain%2Fton%2Fblob%2F5c392e0f2d946877bb79a09ed35068f7b0bd333a%2Fcrypto%2Fblock%2Fblock.tlb%23L144-L146&style=atom-one-dark&type=code&showBorder=on&showLineNumbers=on&showFileMeta=on&showFullPath=on&showCopy=on"></iframe>

    `StateInit` serves to delivery inital data to contract and used in contract deployment. The first field is `split_depth`, of type `(# 5)`. `# 5` means a 5-bit integer. For more, have a look at [StateInit TL-B scheme](https://docs.ton.org/develop/data-formats/msg-tlb#stateinit-tl-b). But as of now, `split_depth`, `special` and `library` are unused. `code` is contract's serialized code, and `data` is contract's initial data.

3. `body:(Either X ^X)`

    Notice that the first line of the scheme has `message$_ {X:Type}`. This is a parametrized type. We use this to mean the type of `X` can be determined at the time of using the type. When it is used to denote the type of another type, it can be used as `(TypeName concreteType)`, like `(MessageRelaxed Any)` or `(MessageRelaxed uint32)`:

    <iframe frameborder="0" scrolling="no" style="width:100%; height:100px;" allow="clipboard-write" src="https://emgithub.com/iframe.html?target=https%3A%2F%2Fgithub.com%2Fton-blockchain%2Fton%2Fblob%2F5c392e0f2d946877bb79a09ed35068f7b0bd333a%2Fcrypto%2Fblock%2Fblock.tlb%23L381&style=atom-one-dark&type=code&showBorder=on&showLineNumbers=on&showFileMeta=on&showFullPath=on&showCopy=on"></iframe>

    If you have been following carefully, you should see that `body:(Either X ^X)` means a type of `X`, or a reference to a cell containing type `X`.

### Message in `mint_tokens`

Now, let's go back to the structure of message in `mint_tokens`:

<iframe frameborder="0" scrolling="no" style="width:100%; height:331px;" allow="clipboard-write" src="https://emgithub.com/iframe.html?target=https%3A%2F%2Fgithub.com%2Fton-blockchain%2Ftoken-contract%2Fblob%2F21e7844fa6dbed34e0f4c70eb5f0824409640a30%2Fft%2Fjetton-minter.fc%23L29-L40&style=atom-one-dark&type=code&showBorder=on&showLineNumbers=on&showFileMeta=on&showFullPath=on&showCopy=on"></iframe>

First, `0x18` is `0b011000`, so we know that this message has `int_msg_info$0` as the constructor of `CommonMsgInfoRelaxed` because it starts with `0`, while others start with `10` and `11` (`ext_in_msg_info$10` and `ext_out_msg_info$11`). And most importantly, `int_msg_info` means 'internal' message, which is a message to be sent in between contracts only. 

```ts
int_msg_info$0 ihr_disabled:Bool bounce:Bool bounced:Bool
  src:MsgAddress dest:MsgAddressInt 
  value:CurrencyCollection ihr_fee:Grams fwd_fee:Grams
  created_lt:uint64 created_at:uint32 = CommonMsgInfoRelaxed;
```

Then:
1. the second leftmost bit is `1`, which means `ihr_disabled` is `bool_true$1`. 
1. The next bit is `1`, meaning `bounce` is `bool_true$1`. 
1. The next bit is `0`, meaning `bounced` is `bool_false$0`.
1. The next two bits are `00`, meaning `addr_none$00 = MsgAddressExt;` is used, because [`_ _:MsgAddressExt = MsgAddress;`](https://github.com/ton-blockchain/ton/blob/5c392e0f2d946877bb79a09ed35068f7b0bd333a/crypto/block/block.tlb#L110C1-L110C32).
1. `store_slice(to_wallet_address)` means we are storing `dest:MsgAddressInt`.
1. `store_coins(amount)` means we are storing `CurrencyCollection`. Recall that `ExtraCurrencyCollection` is not used, so we only care about storing `Grams`. `amount` should actually be less than `120` bits long integer, [according to the spec](https://docs.ton.org/develop/func/stdlib#store_coins).
1. `.store_uint(4 + 2 + 1, 1 + 4 + 4 + 64 + 32 + 1 + 1 + 1)` means the following:
    1. First of all, the uint value to be stored is `4 + 2 + 1 = 0b111`. Given the length of 108, it would actually look like this: 
        `000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000111`
    1. The first 1 bit means empty `ExtraCurrencies` dictionary; `0` stands for empty dictionary bit.
    1. The next two 4-bit long fields are `ihr_fee` and `fwd_fee`. Full of 0.
    1. The 64-bit field is `created_lt`. Full of 0.
    1. The 32-bit field is `created_at`. Full of 0.
    1. The next 1 bit is for the `init` field, because we are done with `CommonMsgInfoRelaxed` and at `init:(Maybe (Either StateInit ^StateInit))`. And recall that when `Maybe` starts with 0, it means there is nothing, and if 1, there is something. In this case, we have our first 1. After that, we also have another 1, which means `StateInit` is stored in another cell.
    1. The next 1 bit is for `body:(Either X ^X)`. Recall that if the bit is 0, it means we're storing `X` in the current cell. If 1, in a reference to another cell. This bit is also 1, so we are pointing to another cell too.
1. `.store_ref(state_init)` stores a reference to another cell of `state_init`. This makes sense because the first two bits of `11` tell us that we are storing `StateInit` in a reference.
1. `.store_ref(master_msg)` stores a reference to another cell of `master_msg`. This makes sense because the last bit of `1` tells us that we are storing `^X`, not `X`.

Lastly, `send_raw_message(msg.end_cell(), 1);` sends the message off. The second parameter is `mode`; `1` means paying transfer fees separately from the message value.

### Burn notification

Now that we finally covered `op::mint()`, let's look at `if (op == op::burn_notification()) {...}`:

<iframe frameborder="0" scrolling="no" style="width:100%; height:499px;" allow="clipboard-write" src="https://emgithub.com/iframe.html?target=https%3A%2F%2Fgithub.com%2Fton-blockchain%2Ftoken-contract%2Fblob%2F21e7844fa6dbed34e0f4c70eb5f0824409640a30%2Fft%2Fjetton-minter.fc%23L72-L91&style=atom-one-dark&type=code&showBorder=on&showLineNumbers=on&showFileMeta=on&showFullPath=on&showCopy=on"></iframe>

First, we obtain `jetton_amoutn` and `from_address` the same way we do for the mint.

Next, we have `throw_unless`:

```ts
throw_unless(74,
    equal_slices(
      calculate_user_jetton_wallet_address(
        from_address, 
        my_address(), 
        jetton_wallet_code
      ), 
      sender_address
    )
);
```

Let's look back at the error code from `JettonConstants.ts`:

<iframe frameborder="0" scrolling="no" style="width:100%; height:100px;" allow="clipboard-write" src="https://emgithub.com/iframe.html?target=https%3A%2F%2Fgithub.com%2Fton-blockchain%2Ftoken-contract%2Fblob%2F21e7844fa6dbed34e0f4c70eb5f0824409640a30%2Fwrappers%2FJettonConstants.ts%23L19&style=atom-one-dark&type=code&showBorder=on&showLineNumbers=on&showFileMeta=on&showFullPath=on&showCopy=on"></iframe>

74 equals `unauthorized_burn`. So we learn that the code checks if `sender_address` is an authorized address by comparing `sender_address` against the expected wallet address of this particular jetton.

The reason that we compare is that we want to make sure `from_address`, which can be manipulated by the sender, gives the same address when put into `calculate_user_jetton_wallet_address`.

Next, `save_data(total_supply - jetton_amount, admin_address, content, jetton_wallet_code);` is to update the total supply because `total_supply` is decreasing by `jetton_amount`.

Next, we load `response_address` by writing `slice response_address = in_msg_body~load_msg_addr();`.

Recall that a `MsgAddress` can be [`addr_none$00` constructor](https://github.com/ton-blockchain/ton/blob/5c392e0f2d946877bb79a09ed35068f7b0bd333a/crypto/block/block.tlb#L100). That's why we are checking `if (response_address.preload_uint(2) != 0)`.

Then, we are building a message again. Let's break it down:

```js
int_msg_info$0 ihr_disabled:Bool bounce:Bool bounced:Bool
  src:MsgAddressInt dest:MsgAddressInt 
  value:CurrencyCollection ihr_fee:Grams fwd_fee:Grams
  created_lt:uint64 created_at:uint32 = CommonMsgInfo;
```

1. `.store_uint(0x10, 6)`. `0x10` in 6 bits = `0b010000`. 
    1. The leftmost bit is `0`, so we know that it's `int_msg_info$0`. 
    1. The next bit is `1`, it means `ihr_disabled` is `true`. At the time of writing, hypercube writing is always disabled.  
    1. Next bit is `0`. Means the message shouldn't be `bounce`d if there are errors during processing.
    1. Next bit is also `0`. Means the message itself is not a result of bouncing.
    1. The next two bits are `00`, meaning `src` is `addr_none$00`.
1. `.store_slice(response_address)` stores `dest:MsgAddressInt`.
1. `.store_coins(0)` means storing nothing for `grams:Grams` part of `CurrencyCollection`.
1. `.store_uint(0, 1 + 4 + 4 + 64 + 32 + 1 + 1)` stores a zero that is `1 + 4 + 4 + 64 + 32 + 1 + 1 = 107` bits long.
    1. The first bit denotes storing nothing for `other:ExtraCurrencyCollection` part of `CurrencyCollection` (empty dictionary).
    1. The next double four bits denote zero `ihr_fee` and `fwd_fee`. These will be overwritten.
    1. The 64 bits are for `created_lt`, which is overwritten.
    1. The 32 bits are for `created_at`, which is also overwritten.
    1. The next 1 bit of zero means there is no `init field; recall the type `init:(Maybe (Either StateInit ^StateInit))`, and zero means there's nothing in `Maybe`.
    1. The next 1 bit  of zero means the body is directly serialized in the current cell, which follows custom layout.
1. The rest of the layout follows [the typical structure of internal message body](https://docs.ton.org/develop/smart-contracts/guidelines/internal-messages#internal-message-body), which is to store 32-bit uint `op` and then 64-bit uint `query_id`:

    ```js
    .store_uint(op::excesses(), 32)
    .store_uint(query_id, 64);
    ```
1. Then the message is sent with a specific mode and flag: `send_raw_message(msg.end_cell(), 2 + 64);`. 2 means "Ignore some errors arising while processing this message during the action phase", and 64 means "Carry all the remaining value of the inbound message in addition to the value initially indicated in the new message". For more, have a look at [the document on message modes](https://docs.ton.org/develop/smart-contracts/messages#message-modes).

And you might be wondering, why is there no code to reduce the balance of the `sender_address`? This is because the burn is already done from [`jetton-wallet.fc`](https://github.com/ton-blockchain/token-contract/blob/21e7844fa6dbed34e0f4c70eb5f0824409640a30/ft/jetton-wallet.fc#L163). That is specifically why this operation is called `op::burn_notification()`, because the message comes from [`jetton-wallet.fc`](https://github.com/ton-blockchain/token-contract/blob/21e7844fa6dbed34e0f4c70eb5f0824409640a30/ft/jetton-wallet.fc#L163) after it burns its own balance. We will look at how the wallet works in a second.

### Admin operations

The rest of the operations are pretty simple:

```js
if (op == 3) { ;; change admin
    throw_unless(
      73, 
      equal_slices(sender_address, admin_address)
    );
    slice new_admin_address = in_msg_body~load_msg_addr();
    save_data(
      total_supply, 
      new_admin_address, 
      content, 
      jetton_wallet_code
    );
    return ();
}

if (op == 4) { ;; change content, delete this for immutable tokens
    throw_unless(
      73, 
      equal_slices(sender_address, admin_address)
    );
    save_data(
      total_supply, 
      admin_address, 
      in_msg_body~load_ref(), 
      jetton_wallet_code
    );
    return ();
}
```

These are just administrative operations that can only be called by `admin_address`. It will update the persistent storage of the contract accordingly.

### Get methods

The last two functions are [get methods](https://docs.ton.org/develop/smart-contracts/guidelines/get-methods), to be called outside of blockchain:

<iframe frameborder="0" scrolling="no" style="width:100%; height:268px;" allow="clipboard-write" src="https://emgithub.com/iframe.html?target=https%3A%2F%2Fgithub.com%2Fton-blockchain%2Ftoken-contract%2Fblob%2F21e7844fa6dbed34e0f4c70eb5f0824409640a30%2Fft%2Fjetton-minter.fc%23L109-L117&style=atom-one-dark&type=code&showBorder=on&showLineNumbers=on&showFileMeta=on&showFullPath=on&showCopy=on"></iframe>

The methods are very self-explanatory. `get_jetton_data` returns the data stored on the persistent storage of the jetton minter (parent) contract. `get_wallet_address` returns the address of user's jetton wallet based on the user's address.

# References

[^1]: [[TON docs] Message layout](https://docs.ton.org/develop/smart-contracts/messages#message-layout)

- [[TON Blog] How to shard your TON smart contract and why - studying the anatomy of TON's Jettons](https://blog.ton.org/how-to-shard-your-ton-smart-contract-and-why-studying-the-anatomy-of-tons-jettons)
- [[TON Blog] Six unique aspects of TON Blockchain that will surprise Solidity developers](https://blog.ton.org/six-unique-aspects-of-ton-blockchain-that-will-surprise-solidity-developers)
- [[Excalidraw] Contracts design diagram](https://excalidraw.com/#json=G8O---P5YSw45M_Fv9rzW,zU2SGCurOzwfre59EImQtQ)
- [[Github] awesome-ton-smart-contracts](https://github.com/dkeysil/awesome-ton-smart-contracts)
- [[Github] `block.tlb`](https://github.com/ton-blockchain/ton/blob/master/crypto/block/block.tlb#L381)
- [[Github] `jetton-wallet.fc`](https://github.com/ton-blockchain/token-contract/blob/main/ft/jetton-wallet.fc)
- [[Github] `jetton-minter.fc`](https://github.com/ton-blockchain/token-contract/blob/main/ft/jetton-minter.fc)
- [[Youtube] Technical Demo: Sharded Smart Contract Architecture for Smart Contract Developers](https://youtu.be/svOadLWwYaM)
- [[PDF] TVM Whitepaper](https://docs.everscale.network/tvm.pdf)