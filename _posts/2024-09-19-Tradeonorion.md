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

First, he deposited 10,000,000 of ORN to the account and called the `lockStake` function to lock and staking the tokens. So the `assetBalances` values of this was reduced to 0.

After that he used the `redeemAtomic` function to transfer the same amount of ORN to one other controlled account called `B`. This functions could allow users to incur liabilities when transferring token through `setLiability` mechanism. When user transfer an amount of token larger than his asset balance, the system would try to check if he has sufficient balance and authorization to deposit the required amount of tokens. In the case of after balance still less than 0, the system would set liability for that user. This mechanism is ensured because the system has one checking mechanism by calculating the total balance includes collateral assets and liability assets.

![_Updatebalance mechanism to transfer token in redeemAtomic function](/assets/images/hacking/tradeonorion/updatebalance.png)

Returning to the real case, the attacker tranfered from `0xf7a8c237ac04c1c3361851ea78e8f50b04c76152` which had balanace 0, so the system set liability of this address with amount 10,000,000 and this account's balanace was -10,000,000.

In the next step, attacker called `requestReleaseStake` to take all the staking tokens and add those token to the `0xf7a8c237ac04c1c3361851ea78e8f50b04c76152` balance. This let this account asset balance to 0 again. Then he called `redeemAtomic` one more time to transfer 10,000,000 more ORN to `B` address. This action manipulated the liabilities array of `0xf7a8c237ac04c1c3361851ea78e8f50b04c76152` to 2 elements have the same value: `["ORN", timestamp, 10,000,000]`

![After first manipulated](/assets/images/hacking/tradeonorion/1.png)


The attacker deposited more 20,000,000 ORN and did all steps one more time. At this time, the liabilities array of `0xf7a8c237ac04c1c3361851ea78e8f50b04c76152` had 3 `["ORN", timestamp, 10,000,000]` elements and the asset balance of this account was only -10,000,000.

![After second manipulated](/assets/images/hacking/tradeonorion/2.png)

With a manipulated liabilities array, attacker used `PancakeSwap`'s flash loan and transferd a huge amount from account `B` back to `0xf7a8c237ac04c1c3361851ea78e8f50b04c76152`. So the balance changed from -10,000,000 to ~1.96 million. Combine both informations, the exploiter can borrow more assets from the system than deserved. Attacker used `redeemAtomic` function again to transfer tokens to attack contract. As mentioned above, there is a checking mechanism to ensure that total balance of user (include asset balance and staking balance) minus liability balance must be greater than 0.

The system first calculates asset balance of user through `calcAssets` function. After that, It calls `calcLiabilities` to sum all liabilities of user and calculates final state by performing subtraction of those values. Only `POSITIVE` state can be accepted to transfer tokens.

![Healthcheck](/assets/images/hacking/tradeonorion/calcPo.png)

Since user's asset balance was ~1.96 million and liabilities array was still exists, the value of `liabilityValue`, which was supposed to be negative, is now positive. Therefore, attacker could bypass the healthcheck mechanism when borrow other tokens. In addition, by manipulating liabilities array, attacker could borrow a greater value of tokens.

![the value change when liabilities array being manipulated](/assets/images/hacking/tradeonorion/calsLia.jpeg)

He used this attack pattern to transfer and withdraw approximately $645,000 in ORN, BNB, BUSD-T token,...

![Root cause](/assets/images/hacking/tradeonorion/hacked.png)

After exploit, the attacker sendback the loan borrowed in `PancakeSwap` flash loan.

![Root cause](/assets/images/hacking/tradeonorion/back.png)


## Root cause

Root cause of the vulnerability is the victim  didn't manage liability correctly. Liability must be updated when user's asset balance changes. But in `requestReleaseStake`,the lack of update liabilities for a user when releasing assets is a key factor contributing to this exploit.

![Root cause](/assets/images/hacking/tradeonorion/requestReleaseStake.png)

In addition, the system should check if user have different liabilities of same token when update or remove liabilities.

![Remove liability](/assets/images/hacking/tradeonorion/removeLiability.png)



## Conclusion

In conclusion, the lack of effective control over liabilities during the balance update process can lead to significant financial discrepancies and user dissatisfaction.  It is essential to implement robust controls that monitor and manage liability levels in real-time, ensuring that users maintain a sustainable balance and preventing financial instability within the platform. By addressing these concerns, we can foster a more secure and trustworthy environment for all users.
