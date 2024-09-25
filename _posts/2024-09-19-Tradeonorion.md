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

This project implements a token exchange and swap platform, utilizing secure and standardized smart contracts. Key contracts like `Exchange.sol`, `ExchangeWithAtomic.sol`, and `ExchangeWithGenericSwap.sol` handle the execution of token trades and swaps, including atomic swaps to ensure transaction integrity. Supporting libraries such as `LibAtomic.sol` and `SafeTransferHelper.sol` provide secure transfer and computation functionalities. The `OrionVault.sol` contract manages asset storage, while `PriceOracleInterface.sol` interacts with price oracles to fetch market rates.

## Exploit analysis

The attack appears to have followed a straightforward pattern: the attacker meticulously manipulated the liabilities array of an account which has address `0xf7a8c237ac04c1c3361851ea78e8f50b04c76152` through a series of preparatory transactions. 

![Pretransaction setup](/assets/images/hacking/tradeonorion/pretx.png)

First, he deposited 10,000,000 ORN into the account and called the `lockStake` function to lock and stake the tokens, which reduced the assetBalances value to 0.

After that, he used the `redeemAtomic` function to transfer the same amount of ORN to another controlled account called `B`. This function allows users to incur liabilities when transferring tokens through the `setLiability` mechanism. When a user transfers an amount of tokens larger than their asset balance, the system checks if they have sufficient balance and authorization to deposit the required amount of tokens. If the balance remains less than 0, the system sets a liability for that user. This mechanism is ensured by the system's check, which calculates the total balance, including collateral assets and liability assets.

![_Updatebalance mechanism to transfer token in redeemAtomic function](/assets/images/hacking/tradeonorion/updatebalance.png)

Returning to the real case, the attacker transferred from `0xf7a8c237ac04c1c3361851ea78e8f50b04c76152`, which had a balance of 0. As a result, the system set a liability for this address with an amount of 10,000,000, causing the account's balance to become -10,000,000.

In the next step, the attacker called `requestReleaseStake` to withdraw all the staked tokens and added those tokens to the balance of `0xf7a8c237ac04c1c3361851ea78e8f50b04c76152`, bringing this account's asset balance back to 0. He then called `redeemAtomic` once more to transfer an additional 10,000,000 ORN to the `B` address. This action manipulated the liabilities array of `0xf7a8c237ac04c1c3361851ea78e8f50b04c76152` to contain two elements with the same value: `["ORN", timestamp, 10,000,000]`.

![After first manipulated](/assets/images/hacking/tradeonorion/1.png)


The attacker deposited an additional 20,000,000 ORN and repeated all the steps once more. At this point, the liabilities array of `0xf7a8c237ac04c1c3361851ea78e8f50b04c76152` contained three elements: `["ORN", timestamp, 10,000,000]`, while the asset balance of this account was only -10,000,000.

![After second manipulated](/assets/images/hacking/tradeonorion/2.png)

With a manipulated liabilities array, the attacker used `PancakeSwap`'s flash loan to transfer a large amount from account `B` back to `0xf7a8c237ac04c1c3361851ea78e8f50b04c76152`. This caused the balance to change from -10,000,000 to approximately 1.96 million. Combined with the earlier information, the exploiter could borrow more assets from the system than they deserved. The attacker then used the `redeemAtomic` function again to transfer tokens to the attack contract.

As mentioned above, there is a checking mechanism to ensure that the total balance of the user (including asset balance and staking balance) minus the liability balance must be greater than 0.

The system first calculates the user's asset balance through the `calcAssets` function. After that, it calls `calcLiabilities` to sum all of the user's liabilities and calculates the final state by performing a subtraction of those values. Only a `POSITIVE` state can be accepted to transfer tokens.

![Healthcheck](/assets/images/hacking/tradeonorion/calcPo.png)

Since the user's asset balance was approximately 1.96 million and the liabilities array still existed, the value of `liabilityValue`, which was supposed to be negative, became positive. Therefore, the attacker could bypass the health check mechanism when borrowing other tokens. Additionally, by manipulating the liabilities array, the attacker could borrow a greater value of tokens.

![the value change when liabilities array being manipulated](/assets/images/hacking/tradeonorion/calsLia.jpeg)

He used this attack pattern to transfer and withdraw approximately $645,000 in ORN, BNB, BUSD-T tokens, and others.

![Root cause](/assets/images/hacking/tradeonorion/hacked.png)

After the exploit, the attacker sent back the loan borrowed from the `PancakeSwap` flash loan.

![Root cause](/assets/images/hacking/tradeonorion/back.png)


## Root cause

The root cause of the vulnerability is that the victim didn't manage liabilities correctly. Liabilities must be updated when the user's asset balance changes. However, in `requestReleaseStake`, the failure to update the liabilities for a user when releasing assets is a key factor contributing to this exploit.

![Root cause](/assets/images/hacking/tradeonorion/requestReleaseStake.png)

In addition, the system should check if a user has different liabilities for the same token when updating or removing liabilities.

![Remove liability](/assets/images/hacking/tradeonorion/removeLiability.png)

## Conclusion

In conclusion, the lack of effective control over liabilities during the balance update process can lead to significant financial discrepancies and user dissatisfaction.  It is essential to implement robust controls that monitor and manage liability levels in real-time, ensuring that users maintain a sustainable balance and preventing financial instability within the platform. By addressing these concerns, we can foster a more secure and trustworthy environment for all users.
