+++
title = "How Jettons work on TON with sharding in mind (Part 2)"
date = 2024-07-19
authors = ["9oelm"]
sort_by = "date"

[extra]
katex_enable = true
+++

## jetton-wallet

Now, we have a look at [`jetton-wallet.fc`](https://github.com/ton-blockchain/token-contract/blob/21e7844fa6dbed34e0f4c70eb5f0824409640a30/ft/jetton-wallet.fc#L1), which is the 'child' contract of jetton.

Let us start with the TL-B scheme of storage:

<iframe frameborder="0" scrolling="no" style="width:100%; height:163px;" allow="clipboard-write" src="https://emgithub.com/iframe.html?target=https%3A%2F%2Fgithub.com%2Fton-blockchain%2Ftoken-contract%2Fblob%2F21e7844fa6dbed34e0f4c70eb5f0824409640a30%2Fft%2Fjetton-wallet.fc%23L24-L27&style=atom-one-dark&type=code&showBorder=on&showLineNumbers=on&showFileMeta=on&showFullPath=on&showCopy=on"></iframe>

The storage contains `balance`, `owner_address`, `jetton_master_address`, and `jetton_wallet_code`. Should be self-explanatory. Recall that `jetton_wallet_code` is the compiled code of `jetton-wallet`.

`load_data()` loads all information stored:

<iframe frameborder="0" scrolling="no" style="width:100%; height:163px;" allow="clipboard-write" src="https://emgithub.com/iframe.html?target=https%3A%2F%2Fgithub.com%2Fton-blockchain%2Ftoken-contract%2Fblob%2F21e7844fa6dbed34e0f4c70eb5f0824409640a30%2Fft%2Fjetton-wallet.fc%23L29-L32&style=atom-one-dark&type=code&showBorder=on&showLineNumbers=on&showFileMeta=on&showFullPath=on&showCopy=on"></iframe>

`save_data` does the opposite:

<iframe frameborder="0" scrolling="no" style="width:100%; height:142px;" allow="clipboard-write" src="https://emgithub.com/iframe.html?target=https%3A%2F%2Fgithub.com%2Fton-blockchain%2Ftoken-contract%2Fblob%2F21e7844fa6dbed34e0f4c70eb5f0824409640a30%2Fft%2Fjetton-wallet.fc%23L34-L36&style=atom-one-dark&type=code&showBorder=on&showLineNumbers=on&showFileMeta=on&showFullPath=on&showCopy=on"></iframe>

`pack_jetton_wallet_data` does nothing but to store the data into a cell, which was the same for `jetton-minter`. Recall that this is because we can only store a cell:

<iframe frameborder="0" scrolling="no" style="width:100%; height:247px;" allow="clipboard-write" src="https://emgithub.com/iframe.html?target=https%3A%2F%2Fgithub.com%2Fton-blockchain%2Ftoken-contract%2Fblob%2F21e7844fa6dbed34e0f4c70eb5f0824409640a30%2Fft%2Fjetton-utils.fc%23L1-L8&style=atom-one-dark&type=code&showBorder=on&showLineNumbers=on&showFileMeta=on&showFullPath=on&showCopy=on"></iframe>

Now, let us start from `recv_internal`, which is invoked when the contract receives an internal message:

<iframe frameborder="0" scrolling="no" style="width:100%; height:856px;" allow="clipboard-write" src="https://emgithub.com/iframe.html?target=https%3A%2F%2Fgithub.com%2Fton-blockchain%2Ftoken-contract%2Fblob%2F21e7844fa6dbed34e0f4c70eb5f0824409640a30%2Fft%2Fjetton-wallet.fc%23L208-L244&style=atom-one-dark&type=code&showBorder=on&showLineNumbers=on&showFileMeta=on&showFullPath=on&showCopy=on"></iframe>

## On bounce

From the previous section explaining `jetton-minter`, we know that the 4th bit is `bounced:Bool`:

<iframe frameborder="0" scrolling="no" style="width:100%; height:163px;" allow="clipboard-write" src="https://emgithub.com/iframe.html?target=https%3A%2F%2Fgithub.com%2Fton-blockchain%2Fton%2Fblob%2F5c392e0f2d946877bb79a09ed35068f7b0bd333a%2Fcrypto%2Fblock%2Fblock.tlb%23L135-L138&style=atom-one-dark&type=code&showBorder=on&showLineNumbers=on&showFileMeta=on&showFullPath=on&showCopy=on"></iframe>

That is why we are checking the 4th bit of the message in the code:

```c++
slice cs = in_msg_full.begin_parse();
int flags = cs~load_uint(4);
if (flags & 1) {
  on_bounce(in_msg_body);
  return ();
}
```

`on_bounce` function is defined here:

<iframe frameborder="0" scrolling="no" style="width:100%; height:289px;" allow="clipboard-write" src="https://emgithub.com/iframe.html?target=https%3A%2F%2Fgithub.com%2Fton-blockchain%2Ftoken-contract%2Fblob%2F21e7844fa6dbed34e0f4c70eb5f0824409640a30%2Fft%2Fjetton-wallet.fc%23L197-L206&style=atom-one-dark&type=code&showBorder=on&showLineNumbers=on&showFileMeta=on&showFullPath=on&showCopy=on"></iframe>

1. `in_msg_body~skip_bits(32); ;; 0xFFFFFFFF`: we do this because [the body of the bounced message will contain 32 bit `0xffffffff` followed by 256 bit from original message](https://docs.ton.org/develop/smart-contracts/guidelines/non-bouncable-messages).
2. we load the data with `load_data()`.
3. the original `op` is retrieved by `int op = in_msg_body~load_uint(32);`. Recall that the structure of the message body is 32 bits of operation followed by 64 bits of query id.
4. `throw_unless(709, (op == op::internal_transfer()) | (op == op::burn_notification()));` throws if the bounced operation isn't internal transfer or burn notification. The error code is `not_enough_ton = 709`.
5. `int query_id = in_msg_body~load_uint(64);` is loaded to get past the 64 bits.
6. `int jetton_amount = in_msg_body~load_coins();` loads the amount that was sent in the message body.
7. The amount is credited again back to balance and written to the storage. This prevents failed messages from falsely deducting the balance. 
    ```ts
    balance += jetton_amount;
    save_data(balance, owner_address, jetton_master_address, jetton_wallet_code);
    ```

We will understand this function better once we look at `send_tokens` and `burn_tokens` functions.

## op::transfer

Next, we prepare arguments to use. Recall the `int_msg_info$0` constructor and its scheme. We are fast forwarding up to the end of `fwd_fee:Grams` in `src:MsgAddressInt dest:MsgAddressInt value:CurrencyCollection ihr_fee:Grams fwd_fee:Grams`:

<iframe frameborder="0" scrolling="no" style="width:100%; height:247px;" allow="clipboard-write" src="https://emgithub.com/iframe.html?target=https%3A%2F%2Fgithub.com%2Fton-blockchain%2Ftoken-contract%2Fblob%2F21e7844fa6dbed34e0f4c70eb5f0824409640a30%2Fft%2Fjetton-wallet.fc%23L219-L226&style=atom-one-dark&type=code&showBorder=on&showLineNumbers=on&showFileMeta=on&showFullPath=on&showCopy=on"></iframe>

`muldiv(cs~load_coins(), 3, 2)` means `floor(cs~load_coins() * 3 / 2)`, meaning it wants to get 1.5 times the original message's `fwd_fee`. `muldiv` is a multiple-then-divide operation. The intermediate result is stored in 513-bit integer, so it won't overflow if the actual result fits into a 257-bit integer. [^1]

The [forward fee](https://docs.ton.org/develop/smart-contracts/fee-calculation#forward-fee) is for outgoing messages (any messages that go out of a contract).

Now when the opcode is `op::transfer`, we call `send_tokens`:

<iframe frameborder="0" scrolling="no" style="width:100%; height:163px;" allow="clipboard-write" src="https://emgithub.com/iframe.html?target=https%3A%2F%2Fgithub.com%2Fton-blockchain%2Ftoken-contract%2Fblob%2F21e7844fa6dbed34e0f4c70eb5f0824409640a30%2Fft%2Fjetton-wallet.fc%23L228-L231&style=atom-one-dark&type=code&showBorder=on&showLineNumbers=on&showFileMeta=on&showFullPath=on&showCopy=on"></iframe>

<iframe frameborder="0" scrolling="no" style="width:100%; height:1108px;" allow="clipboard-write" src="https://emgithub.com/iframe.html?target=https%3A%2F%2Fgithub.com%2Fton-blockchain%2Ftoken-contract%2Fblob%2F21e7844fa6dbed34e0f4c70eb5f0824409640a30%2Fft%2Fjetton-wallet.fc%23L50-L98&style=atom-one-dark&type=code&showBorder=on&showLineNumbers=on&showFileMeta=on&showFullPath=on&showCopy=on"></iframe>

Let's break it down:

1. `int query_id = in_msg_body~load_uint(64);`. Remember we already loaded the opcode, so the next up is 64-bits  long `query_id`.
1. The rest of `in_msg_body` is customizable. We can find the rest of the body at `wrappers/JettonWallet.ts`:

    <iframe frameborder="0" scrolling="no" style="width:100%; height:352px;" allow="clipboard-write" src="https://emgithub.com/iframe.html?target=https%3A%2F%2Fgithub.com%2Fton-blockchain%2Ftoken-contract%2Fblob%2F21e7844fa6dbed34e0f4c70eb5f0824409640a30%2Fwrappers%2FJettonWallet.ts%23L38-L50&style=atom-one-dark&type=code&showBorder=on&showLineNumbers=on&showFileMeta=on&showFullPath=on&showCopy=on"></iframe>

    After `query_id`, `jetton_amount`, specifying the amount to send, is stored by `int jetton_amount = in_msg_body~load_coins();`. Same for `to_owner_address`. The rest of the cell structure is pretty self-explanatory. Just refer to this while reading `send_tokens`.

1. `force_chain(to_owner_address);`

    <iframe frameborder="0" scrolling="no" style="width:100%; height:205px;" allow="clipboard-write" src="https://emgithub.com/iframe.html?target=https%3A%2F%2Fgithub.com%2Fton-blockchain%2Ftoken-contract%2Fblob%2F21e7844fa6dbed34e0f4c70eb5f0824409640a30%2Fft%2Fparams.fc%23L1-L6&style=atom-one-dark&type=code&showBorder=on&showLineNumbers=on&showFileMeta=on&showFullPath=on&showCopy=on"></iframe>

    The reason for calling this function can be traced back to the comment at the top of `jetton-wallet.fc` file:

    <iframe frameborder="0" scrolling="no" style="width:100%; height:100px;" allow="clipboard-write" src="https://emgithub.com/iframe.html?target=https%3A%2F%2Fgithub.com%2Fton-blockchain%2Ftoken-contract%2Fblob%2F21e7844fa6dbed34e0f4c70eb5f0824409640a30%2Fft%2Fjetton-wallet.fc%23L5-L5&style=atom-one-dark&type=code&showBorder=on&showLineNumbers=on&showFileMeta=on&showFullPath=on&showCopy=on"></iframe>

    The TON Blockchain consists of one masterchain and up to $2^32$ workchains. Each workchain is a separate chain with its rules. Each workchain can further split into 260 shardchains, or sub-shards, containing a fraction of the workchain's state. Currently, only one workchain is operating on TON - Basechain [^2]. The Basechain has an `workchain_id` of 0.3
    So the default mode is always transferring within the same workchain; and without changing this line, the jetton will always be forced to be sent to the same workchain.

    [`parse_std_addr`](https://docs.ton.org/develop/func/stdlib#parse_std_addr) returns the workchain id and account id. `int workchain() asm "0 PUSHINT";` just means declaring a zero integer variable. Why zero? because we know that there is only one chain right now, which is the Basechain. And its workchain id is 0. So if `to_owner_address` is from another workchain whose workchain id isn't 0, `throw_unless(333, wc == workchain());` will throw.

1. `(int balance, slice owner_address, slice jetton_master_address, cell jetton_wallet_code) = load_data();`. Load the data.

1. `balance -= jetton_amount;`. Deduct the amount being transferred.

1. `throw_unless(705, equal_slices(owner_address, sender_address));`. Throw if the message didn't come from the owner's wallet.

1. `throw_unless(706, balance >= 0);`. Throw if the balance is insufficient.

1. `cell state_init = calculate_jetton_wallet_state_init(to_owner_address, jetton_master_address, jetton_wallet_code);`. 

    <iframe frameborder="0" scrolling="no" style="width:100%; height:247px;" allow="clipboard-write" src="https://emgithub.com/iframe.html?target=https%3A%2F%2Fgithub.com%2Fton-blockchain%2Ftoken-contract%2Fblob%2F21e7844fa6dbed34e0f4c70eb5f0824409640a30%2Fft%2Fjetton-utils.fc%23L10-L17&style=atom-one-dark&type=code&showBorder=on&showLineNumbers=on&showFileMeta=on&showFullPath=on&showCopy=on"></iframe>

    Don't know why the cell structure looks like this? Back to the TL-B scheme.

    <iframe frameborder="0" scrolling="no" style="width:100%; height:142px;" allow="clipboard-write" src="https://emgithub.com/iframe.html?target=https%3A%2F%2Fgithub.com%2Fton-blockchain%2Fton%2Fblob%2F5c392e0f2d946877bb79a09ed35068f7b0bd333a%2Fcrypto%2Fblock%2Fblock.tlb%23L144-L146&style=atom-one-dark&type=code&showBorder=on&showLineNumbers=on&showFileMeta=on&showFullPath=on&showCopy=on"></iframe>

    1. `store_uint(0, 2)`. Two bits of zeroes mean there's nothing in `split_depth` and `special`. So we go past them.
    1. `store_dict(jetton_wallet_code)`. You might be wondering why `store_dict` instead of something like `store_ref`, because it's a cell? Well, `store_dict` actually looks like `builder store_dict(builder b, cell c) asm(c b) "STDICT";` and stores dictionary `D` represented by cell `c` or `null` into builder `b`. In other words, stores 1-bit and a reference to `c` if `c` is not `null` and 0-bit otherwise. [^3] So it's a perfect use case for short-circuiting `Maybe ^Cell` at once, because `Maybe` requires 1 to be prefixed if there is something in it. For that reason, [`store_maybe_ref`](https://docs.ton.org/develop/func/stdlib#store_maybe_ref) is equivalent to store_dict`.
    1. `.store_dict(pack_jetton_wallet_data(0, owner_address, jetton_master_address, jetton_wallet_code))`
    1. `.store_uint(0, 1)`. This is `library:(Maybe ^Cell)`. 
1. `slice to_wallet_address = calculate_jetton_wallet_address(state_init);`. 

    <iframe frameborder="0" scrolling="no" style="width:100%; height:226px;" allow="clipboard-write" src="https://emgithub.com/iframe.html?target=https%3A%2F%2Fgithub.com%2Fton-blockchain%2Ftoken-contract%2Fblob%2F21e7844fa6dbed34e0f4c70eb5f0824409640a30%2Fft%2Fjetton-utils.fc%23L19-L25&style=atom-one-dark&type=code&showBorder=on&showLineNumbers=on&showFileMeta=on&showFullPath=on&showCopy=on"></iframe>

    `calculate_jetton_wallet_address` takes the result of `calculate_jetton_wallet_state_init` and returns a wallet address. `to_wallet_address` will be `dest:MsgAddressInt` in `int_msg_info$0`, so we will need to refer to `MsgAddressInt` when looking into this.

    <iframe frameborder="0" scrolling="no" style="width:100%; height:163px;" allow="clipboard-write" src="https://emgithub.com/iframe.html?target=https%3A%2F%2Fgithub.com%2Fton-blockchain%2Fton%2Fblob%2F5c392e0f2d946877bb79a09ed35068f7b0bd333a%2Fcrypto%2Fblock%2Fblock.tlb%23L105-L108&style=atom-one-dark&type=code&showBorder=on&showLineNumbers=on&showFileMeta=on&showFullPath=on&showCopy=on"></iframe>

    1. `store_uint(4, 3)`. It stores `0b100`. The first `0b10` is for `addr_std$10` prefix. The next `0` is for `Maybe`, to say that there is none.
    1. `.store_int(workchain(), 8)`. Store a 8-bit long workchain id, which is `0b00000000`. The workchain id is a signed 32-bit integer, but addr_std dictates that `workchain_id` needs to be `int8` in this specific type.
    1. [`cell_hash`](https://docs.ton.org/develop/func/stdlib#cell_hash) returns 256-bit uint hash of a cell. `state_init` is composed of `jetton_wallet_code`, `owner_address`, and `jetton_master_address`, so every jetton wallet address is practically unique, because if it were a different jetton, owner, or a jetton creator, the hash would be different. This is `address:bits256`.

# References

[^1]: [[TON docs] Integer operations](https://docs.ton.org/develop/func/builtins#integer-operations)

[^2]: [[TON docs] Shards](https://docs.ton.org/develop/blockchain/shards)

[^3]: [[TON docs] `store_dict`](https://docs.ton.org/develop/func/stdlib#store_dict)

[[TON Blog] How to shard your TON smart contract and why - studying the anatomy of TON's Jettons](https://blog.ton.org/how-to-shard-your-ton-smart-contract-and-why-studying-the-anatomy-of-tons-jettons)

[[TON Blog] Six unique aspects of TON Blockchain that will surprise Solidity developers](https://blog.ton.org/six-unique-aspects-of-ton-blockchain-that-will-surprise-solidity-developers)

[[Excalidraw] Contracts design diagram](https://excalidraw.com/#json=G8O---P5YSw45M_Fv9rzW,zU2SGCurOzwfre59EImQtQ)

[[Github] awesome-ton-smart-contracts](https://github.com/dkeysil/awesome-ton-smart-contracts)

[[Github] `block.tlb`](https://github.com/ton-blockchain/ton/blob/master/crypto/block/block.tlb#L381)

[[Github] `jetton-wallet.fc`](https://github.com/ton-blockchain/token-contract/blob/main/ft/jetton-wallet.fc)

[[Github] `jetton-minter.fc`](https://github.com/ton-blockchain/token-contract/blob/main/ft/jetton-minter.fc)

[[Youtube] Technical Demo: Sharded Smart Contract Architecture for Smart Contract Developers](https://youtu.be/svOadLWwYaM)
