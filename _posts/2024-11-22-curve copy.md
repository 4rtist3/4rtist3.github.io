---
title: vETH incident with unknown mechanism
tags: smart-contract blockchain
cover: /assets/images/logo/bg.png
---

On November 14, 2024, vETH Token was exploited, resulting in a loss of approximately $477,000 at the time of writing. Let's examine the details of how this attack took place.

Attacker : https://etherscan.io/address/0x351d38733de3f1e73468d24401c59f63677000c9

Attack Contract : https://etherscan.io/address/0x713d2b652e5f2a86233c57af5341db42a5559dd1


Vulnerable Contract : https://etherscan.io/address/0x280a8955a11fcd81d72ba1f99d265a48ce39ac2e

Attack Tx : 

https://etherscan.io/tx/0x90db330d9e46609c9d3712b60e64e32e3a4a2f31075674a58dd81181122352f8

https://etherscan.io/tx/0x1ae40f26819da4f10bc7c894a2cc507cdb31c29635d31fa90c8f3f240f0327c0

https://etherscan.io/tx/0x900891b4540cac8443d6802a08a7a0562b5320444aa6d8eed19705ea6fb9710b

## Exploit analysis

vETH is an ERC20 custom token. It provides features such as token wrapping and unwrapping, allowing users to deposit and withdraw assets, as well as token lending, enabling users to take out loans and repay them. It also has an access control mechanism through a whitelist and factory mechanism, ensuring that only approved users and factories can interact with specific functions. Only the owner can update the whitelist or update trusted factories.

One of the core mechanisms of this token is a loan system that allows valid factory addresses to lend and repay loans to users, while also managing user debt. The `takeLoan` function allows a valid factory to lend a specified amount of virtual tokens to a user. 

![aaa](/assets/images/hacking/veth/takeLong.png)

But there is one unknown mechanism in the Factory contract that can call the `takeLoan` function from external. This function can add liquidity to the Uniswap vETH-(token A) pair using the `takeLoan` function and the user's token A-amounts. After minting an amount of vETH into the Uniswap pair, the pair state is changed and the "constant product" formula is increased. This leads to the attacker earning a free amount of vETH.

On Nov 14, 2024, an attacker used this flow to exploit vETH on 3 Uniswap V2 pairs: `vETH-BIF`, `vETH-Cowbo`, and `vETH-BOVIN`. He gained about $477,000 for this attack. At the time of this writing, the Factory mechanism was private.

![aa2a](/assets/images/hacking/veth/a.png)


## Conclusion

Sharing our insights on the vETH Token exploit highlights the critical need for thorough security audits and robust smart contract design to prevent manipulation, especially in private/internal mechanisms. Effective risk mitigation strategies must be in place to safeguard user funds in DeFi systems, as even minor flaws can result in significant financial losses.