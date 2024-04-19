+++
title = "Lazered #0: Zivoe review"
date = 2024-04-17
authors = ["9oelm"]
sort_by = "date"

[extra]
katex_enable = true
+++

<div
  style="width:100%;display:flex;justify-content:center;align-items:center;"
>
<img src="/blog/lazered-zivoe-review/zivoe-logo.svg" width="50%" />
</div>

Zivoe is a credit protocol where on-chain lenders lend on-chain assets to borrowers whose collaterals are secured off-chain, frequently verified by the Zivoe team.

## Preceding audits

Zivoe has undergone an audit with Runtime Verification in late 2023. It might be a good idea to visit them for a better understanding of the protocol:
- [Core contracts audit](/blog/lazered-zivoe-review/Zivoe_Core_Contracts.pdf)
- [Locker contracts audit](/blog/lazered-zivoe-review/Zivoe_Locker_Contracts.pdf)

## Liquidity provision (deposit)

Anyone can deposit liquidity into Zivoe; only stablecoin is allowed.

### Tranche

There are two types of tranches: senior and junior tranche.

|                | Senior Tranche           | Junior Tranche               |
|----------------|-------------------------|------------------------------|
| Yield          | Generally lower yield              | Generally higher yield                 |
| Risk           | Lower risk; Protected from defaults up to the notional value of junior tranche deposits.               | Higher risk; Served as a first-loss capital in the event of defaults                  |
| tranche token           |  `zSTT` (Zivoe senior tranche)              | `zJTT` (Zivoe junior tranche)         |

Upon depositing into the tranche, the protocol will mint tranche tokens to represent their provision status. Deposit and tranche tokens are always 1:1. The deposit functions are `depositJunior` and `depositSenior` respectively:

<script src="https://emgithub.com/embed-v2.js?target=https%3A%2F%2Fgithub.com%2Fsherlock-audit%2F2024-03-zivoe%2Fblob%2Fd4111645b19a1ad3ccc899bea073b6f19be04ccd%2Fzivoe-core-foundry%2Fsrc%2FZivoeTranches.sol%23L264-L315&style=atom-one-dark&type=code&showBorder=on&showLineNumbers=on&showFileMeta=on&showFullPath=on&showCopy=on"></script>

Burning a tranche token will reclaim the original deposit. Staking it with `stake` will earn an additional yield (we will come to the explanation of the code later):

<script src="https://emgithub.com/embed-v2.js?target=https%3A%2F%2Fgithub.com%2Fsherlock-audit%2F2024-03-zivoe%2Fblob%2Fd4111645b19a1ad3ccc899bea073b6f19be04ccd%2Fzivoe-core-foundry%2Fsrc%2FZivoeRewards.sol%23L251-L262&style=atom-one-dark&type=code&showBorder=on&showLineNumbers=on&showFileMeta=on&showFullPath=on&showCopy=on"></script>

This could be a realistic example of the impact of defaults on tranches: let's say the notional value of the senior tranche is 10M USD, and that of the junior tranche 5M USD. And then if there is a loss of 4M USD due to defaults on the loans, the junior tranche investors fully absorb the impact by losing 4M USD. However, the senior tranche depositors are not impacted by the defaults at all and their deposits are still intact due to the junior tranche absorbing the defaults.

## Borrowing

A prospective borrower would need to go through an application process including KYC and credit evaluation. Then, the Zivoe team would offer a loan to the borrower based on his credit health.

Borrowers are expected to make repayments according to the prearranged schedule. Zivoe's smart contracts handle the accounting.

It's also possible to refinance the loan by making another application that needs to be approved. The loan term has to be updated. Multiple loans will be merged into a single loan to simplify repayment.

### Types of fixed-term loan

Zivoe supports two types of a fixed loan:

1. Bullet loan: regular interests payments are made throughout the term of the loan, and the entire principal is paid at the maturity date.
2. Amortizing loan: parts of the principal and interests are regularly repaid.

The payment schedule is represented by `paymentSchedule` of `struct Loan`:

<script src="https://emgithub.com/embed-v2.js?target=https%3A%2F%2Fgithub.com%2Fsherlock-audit%2F2024-03-zivoe%2Fblob%2Fd4111645b19a1ad3ccc899bea073b6f19be04ccd%2Fzivoe-core-foundry%2Fsrc%2Flockers%2FOCC%2FOCC_Modular.sol%23L111-L111&style=atom-one-dark&type=code&showBorder=on&showLineNumbers=on&showFileMeta=on&showFullPath=on&showCopy=on"></script>

Below is a very simple graph illustrating different types of fixed-term loans.

![Fixed term loan comparison](/blog/lazered-zivoe-review/fixed_loan_comparison.png)

Zivoe supports various durations and payment frequencies in these two types of loans.

### Fee

There isn't a 'borrowing fee'. Only interest rate exists to put an extra burden on the borrower. 

However, late fee interest rate is added upon the original interest rate if a payment is missed. Late fee interest only begins to accrue on the outstanding principal. When the missed payments are paid off, the late fee interest is removed.

### Defaults

#### Delinquent loan

When a loan payment is missed, late fees immediately start accruing on the outstanding balance of the loan. This loan is called 'delinquent'. A limited grace period is given for the missed repayment.

To resolve a delinquent loan, the borrower must supply late fees in addition to the principal and interest due on the missed payment

#### Defaulted loan

Loans are marked defaulted when a borrower hasn't made timely repayments within the grace period.

Zivoe tracks the value of defaulted loans with a global variable `defaults`:

<script src="https://emgithub.com/embed-v2.js?target=https%3A%2F%2Fgithub.com%2Fsherlock-audit%2F2024-03-zivoe%2Fblob%2Fd4111645b19a1ad3ccc899bea073b6f19be04ccd%2Fzivoe-core-foundry%2Fsrc%2FZivoeGlobals.sol%23L45-L45&style=atom-one-dark&type=code&showBorder=on&showLineNumbers=on&showFileMeta=on&showFullPath=on&showCopy=on"></script>

This will lower the adjusted supply of the junior tranche first, and then if it exceeds the notional value of the junior tranche, the adjusted supply of the senior tranche will decrease accordingly too. This is reflected in the function `adjustedSupplies`:

<script src="https://emgithub.com/embed-v2.js?target=https%3A%2F%2Fgithub.com%2Fsherlock-audit%2F2024-03-zivoe%2Fblob%2Fd4111645b19a1ad3ccc899bea073b6f19be04ccd%2Fzivoe-core-foundry%2Fsrc%2FZivoeGlobals.sol%23L123-L136&style=atom-one-dark&type=code&showBorder=on&showLineNumbers=on&showFileMeta=on&showFullPath=on&showCopy=on"></script>

Adjusted supplies also affect the target return. For example, suppose that we have 100,000 `zSTT` and 10,000 zJTT, each tranche targeting 1% and 3% return.
- Under a normal circumstance without defaults. Then, the expected return is $100,000 \times 1\\% + 10,000 \times 3\\% = 1,300 \text{ USDC}$.
- When there is a default of 2,000 USD (or USDC), the expected return is $100,000 \times 1\\% + 8,000 \times 3\\% = 1,240 \text{ USDC}$, because the junior tranche absorbs the default first.

The default does not decrease the circulating supply of tranche tokens. The only number that is affected by the default is the adjusted supply.

When the defaults are resolved, the global variable `defaults` will be decreased and the adjusted supply of tranches will be recovered to the level close to (or the same as) the circulating supply.

### Loan workflow

The loan is managed by `OCC_Modular.sol` contract, which stands for on-chain credit. The first step that happens on-chain related to any loan is to create a loan offer:

<script src="https://emgithub.com/embed-v2.js?target=https%3A%2F%2Fgithub.com%2Fsherlock-audit%2F2024-03-zivoe%2Fblob%2Fd4111645b19a1ad3ccc899bea073b6f19be04ccd%2Fzivoe-core-foundry%2Fsrc%2Flockers%2FOCC%2FOCC_Modular.sol%23L531-L570&style=atom-one-dark&type=code&showBorder=on&showLineNumbers=on&showFileMeta=on&showFullPath=on&showCopy=on"></script>

Upon meeting all input validations, `Loan` will be created. `Loan` is the following struct:

<script src="https://emgithub.com/embed-v2.js?target=https%3A%2F%2Fgithub.com%2Fsherlock-audit%2F2024-03-zivoe%2Fblob%2Fd4111645b19a1ad3ccc899bea073b6f19be04ccd%2Fzivoe-core-foundry%2Fsrc%2Flockers%2FOCC%2FOCC_Modular.sol%23L99-L113&style=atom-one-dark&type=code&showBorder=on&showLineNumbers=on&showFileMeta=on&showFullPath=on&showCopy=on"></script>

Notice that when the loan is created, `paymentDueBy = 0` meaning that the repayment can happen from this moment, `paymentsRemaining = term`, `offerExpiry = block.timestamp + 3 days`, and `state = LoanState.Offered`.

If the underwriting team changes its mind, it can always call [cancelOffer](https://github.com/sherlock-audit/2024-03-zivoe/blob/d4111645b19a1ad3ccc899bea073b6f19be04ccd/zivoe-core-testing/lib/zivoe-core-foundry/src/lockers/OCC/OCC_Modular.sol#L520-L529), which then converts the state to `LoanState.Cancelled`.

Then, with `state` still being `LoanState.Offered`, the `borrower` accepts the loan:

<script src="https://emgithub.com/embed-v2.js?target=https%3A%2F%2Fgithub.com%2Fsherlock-audit%2F2024-03-zivoe%2Fblob%2Fd4111645b19a1ad3ccc899bea073b6f19be04ccd%2Fzivoe-core-foundry%2Fsrc%2Flockers%2FOCC%2FOCC_Modular.sol%23L459-L487&style=atom-one-dark&type=code&showBorder=on&showLineNumbers=on&showFileMeta=on&showFullPath=on&showCopy=on"></script>

After variable validations, the `stablecoin` of amount `loans[id].principalOwed` is sent to the borrower.

Later on, any address can `makePayment` for an loan of `id`. The `amount` doesn't need to be specified; it is automatically calculated by `amountOwed`.

<script src="https://emgithub.com/embed-v2.js?target=https%3A%2F%2Fgithub.com%2Fsherlock-audit%2F2024-03-zivoe%2Fblob%2Fd4111645b19a1ad3ccc899bea073b6f19be04ccd%2Fzivoe-core-foundry%2Fsrc%2Flockers%2FOCC%2FOCC_Modular.sol%23L572-L602&style=atom-one-dark&type=code&showBorder=on&showLineNumbers=on&showFileMeta=on&showFullPath=on&showCopy=on"></script>

Alternatively, the borrower can choose to pay a loan in full at once:

<script src="https://emgithub.com/embed-v2.js?target=https%3A%2F%2Fgithub.com%2Fsherlock-audit%2F2024-03-zivoe%2Fblob%2Fd4111645b19a1ad3ccc899bea073b6f19be04ccd%2Fzivoe-core-foundry%2Fsrc%2Flockers%2FOCC%2FOCC_Modular.sol%23L489-L518&style=atom-one-dark&type=code&showBorder=on&showLineNumbers=on&showFileMeta=on&showFullPath=on&showCopy=on"></script>

If the loan isn't repaid on time, the underwriting team calls `markDefault`:

<script src="https://emgithub.com/embed-v2.js?target=https%3A%2F%2Fgithub.com%2Fsherlock-audit%2F2024-03-zivoe%2Fblob%2Fd4111645b19a1ad3ccc899bea073b6f19be04ccd%2Fzivoe-core-foundry%2Fsrc%2Flockers%2FOCC%2FOCC_Modular.sol%23L604-L618&style=atom-one-dark&type=code&showBorder=on&showLineNumbers=on&showFileMeta=on&showFullPath=on&showCopy=on"></script>

## Redemptions (withdrawal)

A liquidity provider can withdraw their original stablecoin deposit by burning a tranche token. He needs to follow these steps:

1. Submit a redemption request specifying `uint256 amount` to be redeemed. This stakes the requested tranche tokens into the redemption locker:

<script src="https://emgithub.com/embed-v2.js?target=https%3A%2F%2Fgithub.com%2Fsherlock-audit%2F2024-03-zivoe%2Fblob%2Fd4111645b19a1ad3ccc899bea073b6f19be04ccd%2Fzivoe-core-foundry%2Fsrc%2Flockers%2FOCR%2FOCR_Modular.sol%23L212-L228&style=atom-one-dark&type=code&showBorder=on&showLineNumbers=on&showFileMeta=on&showFullPath=on&showCopy=on"></script>

2. Wait for the next epoch: an epoch is 14 days. The liquidity provider will wait for the next epoch to start to be able to redeem.

    The epoch is ticked automatically whenever relevant actions are called.

<script src="https://emgithub.com/embed-v2.js?target=https%3A%2F%2Fgithub.com%2Fsherlock-audit%2F2024-03-zivoe%2Fblob%2Fd4111645b19a1ad3ccc899bea073b6f19be04ccd%2Fzivoe-core-foundry%2Fsrc%2Flockers%2FOCR%2FOCR_Modular.sol%23L309-L336&style=atom-one-dark&type=code&showBorder=on&showLineNumbers=on&showFileMeta=on&showFullPath=on&showCopy=on"></script>

<script src="https://emgithub.com/embed-v2.js?target=https%3A%2F%2Fgithub.com%2Fsherlock-audit%2F2024-03-zivoe%2Fblob%2Fd4111645b19a1ad3ccc899bea073b6f19be04ccd%2Fzivoe-core-foundry%2Fsrc%2Flockers%2FOCR%2FOCR_Modular.sol%23L155-L159&style=atom-one-dark&type=code&showBorder=on&showLineNumbers=on&showFileMeta=on&showFullPath=on&showCopy=on"></script>

  The modifier `_tickEpoch` follows any mutative functions in `OCR_Modular` (On-chain redemption) contract, making sure the epoch is ticked (updated) before the functions are called.

3. Process the redemption of tranche tokens:

<script src="https://emgithub.com/embed-v2.js?target=https%3A%2F%2Fgithub.com%2Fsherlock-audit%2F2024-03-zivoe%2Fblob%2Fd4111645b19a1ad3ccc899bea073b6f19be04ccd%2Fzivoe-core-foundry%2Fsrc%2Flockers%2FOCR%2FOCR_Modular.sol%23L253-L307&style=atom-one-dark&type=code&showBorder=on&showLineNumbers=on&showFileMeta=on&showFullPath=on&showCopy=on"></script>

the maximum amount of tranche tokens that can be redeemed in each epoch is determeined by the available capital for redemption in the redemption locker. This means that tranche investors may not be able to redeem a full `amount` in the original request. In this case, tranche investors are treated fairly: each of them will only be able to redeem a $\frac{\text{redemption request amount} \times \text{total available capital}_{\text{epoch}}}{sum(\text{outstanding redemption request amount for all requests})}$.

For example, two users submit redemption requests during epoch 1: one for 100,000 zSTT and another for 100,000 zJTT. With 50,000 USDC available in the redemption locker, these users have to wait until epoch 2 to process their requests.

As epoch 2 starts, they can begin redeeming their tokens. However, due to the limited available capital (50,000 USDC), and a total outstanding request size of 200,000 tranche tokens, each user can redeem a maximum of 25,000 USDC. In subsequent epochs, if more capital becomes available, more tranche tokens can be redeemed.

### Redemption in case of defaults

As discussed, the defaults should always be absorbed by the junior tranche first. This is reflected in `tickEpoch` function:

<script src="https://emgithub.com/embed-v2.js?target=https%3A%2F%2Fgithub.com%2Fsherlock-audit%2F2024-03-zivoe%2Fblob%2Fd4111645b19a1ad3ccc899bea073b6f19be04ccd%2Fzivoe-core-foundry%2Fsrc%2Flockers%2FOCR%2FOCR_Modular.sol%23L309-L336&style=atom-one-dark&type=code&showBorder=on&showLineNumbers=on&showFileMeta=on&showFullPath=on&showCopy=on"></script>

When `totalDefaults > zJTTSupply`, this means that the amount of defaults exceeds the notional value of the liquidity deposited in the junior tranche. That means `zJTT` for now is virtually worthless, because it shouldn't be able to redeem any money. Therefore, the discount for the junior tranche at this epoch, reprsented by `epochDiscountJunior`, is set at 100%, which is `BIPS = 10000`. This will also affect the senior tranche as the value of defaults is greater than what the junior tranche can cover. In that case, the discount rate for the senior tranche is: `epochDiscountSenior = (totalDefaults * RAY / (totalDefaults - zJTTSupply)) / 10**23`, because `totalDefaults - zJTTSupply` is the amount that cannot be covered by the junior tranche.

Let's give an example. Assume the supply of tranche tokens is 1,000,000 zSTT and 250,000 zJTT, and then a loan of 125,000 USDC defaults. The impact of this default means that the adjusted tranche token supplies are now 1,000,000 zSTT and 125,000 zJTT, indicating that the junior tranche has lost half of its backed value due to the default. Consequently, junior liquidity providers will encounter a 50% discount on their redemption requests. Let's consider a scenario where a junior liquidity provider requests to redeem 50,000 zJTT. Suppose there's 60,000 USDC in the redemption locker and no other redemption requests are outstanding. Given the 50% default loss, this junior liquidity provider can only redeem 25,000 USDC for his 50,000 zJTT, equivalent to a rate of 0.5 USDC per zJTT. However, if a senior liquidity provider also wishes to redeem their tranche tokens, say 10,000 zSTT, in the same epoch, they would be able to redeem the 10,000 zSTT for a full 10,000 USDC, unaffected by the junior tranche's loss.

On ther other thand, when `totalDefaults <= zTTSupply`, the discount rate for the junior tranche should be $\frac{\text{total defaults}}{\text{liquidity supplied in junior tranche}}$. This is calculated by `epochDiscountJunior = (totalDefaults * RAY / zJTTSupply) / 10**23`. And the senior tranche will not be affected by any means. Let's leave it at that, and we will come to the matter of unit precision later.

## Yield generation and distribution

The yield is generated in two ways:
1. Deploying an unused capital in the tranches into other DeFi protocols
2. Earning interests from the borrowers

The strategies in the deployment of an unused capital are implemented by several lockers prefixed with 'OCY', which stands for 'on-chain yield':
- [OCY_Convex_A.sol](https://github.com/sherlock-audit/2024-03-zivoe/blob/d4111645b19a1ad3ccc899bea073b6f19be04ccd/zivoe-core-foundry/src/lockers/OCY/OCY_Convex_A.sol#L39-L39): allocates stablecoins to the alUSD/FRAXBP meta-pool and stakes the LP tokens on Convex (pool id `106`)
- [OCY_Convex_B.sol](https://github.com/sherlock-audit/2024-03-zivoe/blob/d4111645b19a1ad3ccc899bea073b6f19be04ccd/zivoe-core-foundry/src/lockers/OCY/OCY_Convex_B.sol#L34-L34): allocates stablecoins to the sUSD base-pool and stakes the LP tokens on Convex. (pool id `4`)
- [OCY_Convex_C.sol](https://github.com/sherlock-audit/2024-03-zivoe/blob/d4111645b19a1ad3ccc899bea073b6f19be04ccd/zivoe-core-foundry/src/lockers/OCY/OCY_Convex_C.sol#L34-L34): allocates stablecoins to the PYUSD/USDC base-pool and stakes the LP tokens on Convex (pool id `270`)
- [OCY_OUSD.sol](https://github.com/sherlock-audit/2024-03-zivoe/blob/d4111645b19a1ad3ccc899bea073b6f19be04ccd/zivoe-core-foundry/src/lockers/OCY/OCY_OUSD.sol#L19-L19): escrows OUSD and handles accounting for yield distributions.

The generated yield plus any additional returns known as residual, is distributed to all stakeholders in the protocol:
- ZVE (Zivoe Token) stakers: they are awared a share of revenue adjustable via governance generated by the protocol.
- Zivoe development company: the operating team behind Zivo protocol earns a share of protocol revenue adjustable via governance.
- Senior/junior liquidity providers: they are only awarded the remaining yield after the above two stakeholders are compensated. Both the senior and junior tranche have annual target yields that the protocol aims to deliver. Zivoe's governance system regularly calibrates these targets, ensuring they align with the portfolio's current risk profile. This enables Zivoe to deliver risk-adjusted yield to liquidity providers. 

The distribution algorithm that manages the disbursement of yield to these stakeholders is known as the Yield Distribution Locker ("YDL").

The yield is sent to the YDL when:
- as soon as a borrower makes a payment
- the time reaches the end of a month (30 days)

Then, the yield is converted to USDC, and it is distributed to the aforementioned stakeholders. First, a certain % of protocol revenue is split between ZVE stakers and the Zivo team. Then, the remainder is shared among liquidity providers and resiual participants.

## Initial Tranche Offering (ITO)

Initially, the tranches [`ZivoeTranches` contract will be locked](https://github.com/sherlock-audit/2024-03-zivoe/blob/d4111645b19a1ad3ccc899bea073b6f19be04ccd/zivoe-core-foundry/src/ZivoeTranches.sol#L61-L61), leaving users with the only option to deposit in [`ZivoeITO`](https://github.com/sherlock-audit/2024-03-zivoe/blob/d4111645b19a1ad3ccc899bea073b6f19be04ccd/zivoe-core-foundry/src/ZivoeITO.sol#L91-L91) first.

Essentially, the tranches in ITO works the same way as the tranches in `ZivoeTranches`. The Zivoe team `commence`s the ITO period:

<script src="https://emgithub.com/embed-v2.js?target=https%3A%2F%2Fgithub.com%2Fsherlock-audit%2F2024-03-zivoe%2Fblob%2Fd4111645b19a1ad3ccc899bea073b6f19be04ccd%2Fzivoe-core-foundry%2Fsrc%2FZivoeITO.sol%23L337-L347&style=atom-one-dark&type=code&showBorder=on&showLineNumbers=on&showFileMeta=on&showFullPath=on&showCopy=on"></script>

Then, the same `depositJunior` and `depositSenior` functions will be available.

The only difference is that each deposit now gives you a credit called `juniorCredit` or `seniorCredit`, which tracks the balance of `pZVE` (pre-ZVE):

<script src="https://emgithub.com/embed-v2.js?target=https%3A%2F%2Fgithub.com%2Fsherlock-audit%2F2024-03-zivoe%2Fblob%2Fd4111645b19a1ad3ccc899bea073b6f19be04ccd%2Fzivoe-core-foundry%2Fsrc%2FZivoeITO.sol%23L109-L110&style=atom-one-dark&type=code&showBorder=on&showLineNumbers=on&showFileMeta=on&showFullPath=on&showCopy=on"></script>

Observe that upon deposit, it will increase the credit amount of the user in the corresponding tranche proportional to the amount of stablecoin deposited:

<script src="https://emgithub.com/embed-v2.js?target=https%3A%2F%2Fgithub.com%2Fsherlock-audit%2F2024-03-zivoe%2Fblob%2Fd4111645b19a1ad3ccc899bea073b6f19be04ccd%2Fzivoe-core-foundry%2Fsrc%2FZivoeITO.sol%23L265-L265&style=atom-one-dark&type=code&showBorder=on&showLineNumbers=on&showFileMeta=on&showFullPath=on&showCopy=on"></script>

<script src="https://emgithub.com/embed-v2.js?target=https%3A%2F%2Fgithub.com%2Fsherlock-audit%2F2024-03-zivoe%2Fblob%2Fd4111645b19a1ad3ccc899bea073b6f19be04ccd%2Fzivoe-core-foundry%2Fsrc%2FZivoeITO.sol%23L292-L292&style=atom-one-dark&type=code&showBorder=on&showLineNumbers=on&showFileMeta=on&showFullPath=on&showCopy=on"></script>

Here's a breakdown of received/allocated tokens for deposit in each tranche:

|                | 1 unit of stablecoin deposited in senior tranche           | 1 unit of stablecoin deposited in junior tranche               |
|----------------|-------------------------|------------------------------|
| corresponding amount of received tranche token          | 1 zSTT              | 1 zJTT                 |
| corresponding amount of allocated (vested) pre-ZVE token           | 3 pZVE | 1 pZVE   |

At the end of the ITO, 5% of the total ZVE supply will be airdropped to the depositors based on their proportional ownership of pZVE. The ZVE aidrop is under a 1-year linear vesting schedule.

The proportional ownership of pZVE is calculated as in `claimAirdrop` function:

<script src="https://emgithub.com/embed-v2.js?target=https%3A%2F%2Fgithub.com%2Fsherlock-audit%2F2024-03-zivoe%2Fblob%2Fd4111645b19a1ad3ccc899bea073b6f19be04ccd%2Fzivoe-core-foundry%2Fsrc%2FZivoeITO.sol%23L197-L242&style=atom-one-dark&type=code&showBorder=on&showLineNumbers=on&showFileMeta=on&showFullPath=on&showCopy=on"></script>

In the code above, `upper * middle / lower` represents the proportional ownership of pZVE per user, which is [`amountToVest` parameter in `createVestingSchedule`](https://github.com/sherlock-audit/2024-03-zivoe/blob/d4111645b19a1ad3ccc899bea073b6f19be04ccd/zivoe-core-foundry/src/ZivoeRewardsVesting.sol#L375-L387). Calling `createVestingSchedule` will immediately create a vesting schedule for that user with that specified amount. `amountToVest` could be expressed in this way: 

$$
\text{upper} = (depositedAmount({\text{senior tranche}, \text{user}}) \times 3 \\\ + depositedAmount({\text{junior tranche}, \text{user}}))\\\ 
$$
$$
\text{middle} = \frac{totalSupply(\text{ZVE})}{20} = totalSupply(\text{ZVE}) \times 5\\% \\\ 
$$
$$
\text{lower} = totalSupply(\text{zSTT}) \times 3 + totalSupply(\text{zJTT})
$$
$$
\text{amount to vest} = \frac{upper \times middle}{lower}
$$
