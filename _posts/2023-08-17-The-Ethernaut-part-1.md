---
title: The Ethernaut (part 1)
tags: wargame blockchain
cover: /assets/images/ethernaut/the_ethernaut.png
mode: immersive
header:
  theme: dark
article_header:
  type: cover
  image:
    src: /assets/images/ethernaut/aaa@4x.png
---

## Level 0 - Hello Ethernaut
<p style="text-align:center;">
    <img class="image image--xl" src="/assets/images/ethernaut/lv0@4x.png"/>
</p>

This level is tutorial to The Ethernaut and how to interact with the ABI (Application Binary Interface). As step 9 in the tutorial, we should look into level info by the `await contract.info()`.
```js
>> await contract.info()
"You will find what you need in info1()."
```
Change to `info1()`
```js
>> await contract.info1()
'Try info2(), but with "hello" as a parameter.' 
```
And continue following commands.
```js
>> await contract.info2("hello")
"The property infoNum holds the number of the next info method to call." 
>> (await contract.infoNum()).toString()
"42"
>> await contract.info42()
"theMethodName is the name of the next method."
>> await contract.theMethodName()
"The method name is method7123949."
>> await contract.method7123949()
"If you know the password, submit it to authenticate()." 
```

Now the goal is to find the password and then submit to authenticate. Check again so there is a function call name `password`. Just call it and submit the answer.

```js
>> await contract.password()
"ethernaut0" 
```

Accept the Authenticate transaction in Metamask and done level 0.
## Level 1 - Fallback
This level give the source code.
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Fallback {

  mapping(address => uint) public contributions;
  address public owner;

  constructor() {
    owner = msg.sender;
    contributions[msg.sender] = 1000 * (1 ether);
  }

  modifier onlyOwner {
        require(
            msg.sender == owner,
            "caller is not the owner"
        );
        _;
    }

  function contribute() public payable {
    require(msg.value < 0.001 ether);
    contributions[msg.sender] += msg.value;
    if(contributions[msg.sender] > contributions[owner]) {
      owner = msg.sender;
    }
  }

  function getContribution() public view returns (uint) {
    return contributions[msg.sender];
  }

  function withdraw() public onlyOwner {
    payable(owner).transfer(address(this).balance);
  }

  receive() external payable {
    require(msg.value > 0 && contributions[msg.sender] > 0);
    owner = msg.sender;
  }
}
```
