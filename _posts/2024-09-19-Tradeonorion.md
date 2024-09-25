---
title: TradeOnOrion incident analysis
tags: smart-contract blockchain
cover: /assets/images/logo/bg.png
---

On May 28, 2024, the Orion project was exploited, resulting in a loss of approximately $645,000 at the time of writing. Let's examine the details of how this attack took place.

## Overview

Attacker address: [https://bscscan.com/address/0x51177db1ff3b450007958447946a2eee388288d2](https://bscscan.com/address/0x51177db1ff3b450007958447946a2eee388288d2)

Attack transactions: [https://bscscan.com/tx/0x660837a1640dd9cc0561ab7ff6c85325edebfa17d8b11a3bb94457ba6dcae18c](https://bscscan.com/tx/0x660837a1640dd9cc0561ab7ff6c85325edebfa17d8b11a3bb94457ba6dcae18c)

Attack contract: [https://bscscan.com/address/0xf8bfac82bdd7ac82d3aeec98b9e1e73579509db6](https://bscscan.com/address/0xf8bfac82bdd7ac82d3aeec98b9e1e73579509db6)

Setup address: [https://bscscan.com/address/0xf7a8c237ac04c1c3361851ea78e8f50b04c76152](https://bscscan.com/address/0xf7a8c237ac04c1c3361851ea78e8f50b04c76152)

## Background

(Optional)

## Exploit analysis

The attack appears to have followed a straightforward pattern: the attacker meticulously manipulated the liabilities array of an account which has address `0xf7a8c237ac04c1c3361851ea78e8f50b04c76152` through a series of preparation transactions. 

![Pretransaction setup](/assets/images/hacking/tradeonorion/pretx.png)

First, he deposited 10,000,000 of ORN to the account and called the `lockStake` function to lock and staking the tokens. So the `assetBalances` values of this was redued to 0.

After that he used the `redeemAtomic` function to transfer the same amount of ORN to one other controlled account called `B`. This functions could allow users to incur liabilities when transferring token through `setLiability` mechanism. When user transfer an amount of token larger than his asset balance, the system would try to check if he has sufficient balance and authorization to deposit the required amount of tokens. In the case of after balance still less than 0, the system would set liability for that user. This mechanism is ensured because the system has one checking mechanism by calculating the total balance includes collateral assets and liability assets.

![_Updatebalance mechanism to transfer token in redeemAtomic function](/assets/images/hacking/tradeonorion/updatebalance.png)

Returning to the real case, the attacker tranfered from `0xf7a8c237ac04c1c3361851ea78e8f50b04c76152` which had balanace 0, so the system set liability of this address with amount 10,000,000 and this account's balanace was -10,000,000.

In the next step, attacker called `requestReleaseStake` to take all the staking tokens and add those token to the `0xf7a8c237ac04c1c3361851ea78e8f50b04c76152` balance. This let this account asset balance to 0 again. Then he called `redeemAtomic` one more time to transfer 10,000,000 more ORN to `B` address. This action manipulated the liabilities array of `0xf7a8c237ac04c1c3361851ea78e8f50b04c76152` to 2 elements have the same value: `["ORN", timestamp, 10,000,000]`

(hình ảnh sau bước 1)

The attacker deposited more 20,000,000 ORN and did all steps one more time. At this time, the liabilities array of `0xf7a8c237ac04c1c3361851ea78e8f50b04c76152` had 3 `["ORN", timestamp, 10,000,000]` elements and the asset balance of this account was only -10,000,000.

(Hình ảnh sau bước 2)

With a manipulated liabilities array, attacker used `PancakeSwap`'s flash loan and transferd a huge amount from account `B` back to `0xf7a8c237ac04c1c3361851ea78e8f50b04c76152`. So the balance changed from -10,000,000 to ~198 million. Combine both informations, the exploiter can borrow more assets from the system than deserved. 

## Root cause

![Root cause](/assets/images/hacking/tradeonorion/requestreleasestake.png)


## Conclusion



