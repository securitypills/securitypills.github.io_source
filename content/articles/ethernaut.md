---
title: Ethernaut Challenges
date: 2022-08-03
---
{{<figure src="../images/ethernaut.png">}}
[Ethernaut](https://ethernaut.openzeppelin.com/) is OpenZeppelin Web3/Solidity based wargame to learn about Ethereum smart contract security and become familiar with programming principles in [Solidity](https://docs.soliditylang.org/en/v0.8.15/).

Although the game was launched few years ago it has become a good place to start for those who are interested on security and smart contracts.

Challenges are currently running on the [Rinkeby testnet](https://www.alchemy.com/overviews/rinkeby-testnet) and you will require to use a Rinkey Faucet to get free testnet ETH and test the smart contracts.

This article is my little contribution to those interested on learning the basics on Ethereum smart contract security by providing  guidance and explanations on how to solve each Ethernaut challenge using [brownie framework](https://github.com/eth-brownie/brownie). There are different approaches and frameworks that can be used, rather than the approach you will find here, so feel free to explore and learn from others.

You can find my solutions on my [GitHub repository](https://github.com/0xroot-bf/ethernaut) and a technical detailed explanation for each challenge below.

Let's go over these Ethernaut's challenges and learn few security aspects of smart contracts written in Solidity!

## Ethernaut Challenges

### 0x0. Hello

### 0x1. Fallback
The first real challenge and a simple one actually. You have to become the contract's owner, and you can achieve this by calling the contract's `fallback` function. However, you must send some Ethers first and call the `contribute` function once. After that, you will become the legitimate owner and you will be able to call the `withdraw` method to steal all the funds.

You can read a walkthrough for this challenge [here](../ethernaut-1-fallback).

### 0x2. Fallout
This vulnerability is unlikely to occur on recent versions of Solidity. The problem with this contract is a typo in the `Fal1out` function, which can be called by anyone, thus becoming the contract's owner.

### 0x3. Coin Flip

### 0x4. Telephone

### 0x5. Token

### 0x6. Delegation

### 0x7. Force

### 0x8. Vault

### 0x9. King
This challenge recreates the famous [King of the Ether](https://www.kingoftheether.com/thrones/kingoftheether/index.html) game (actually affected by a [similar vulnerability](http://www.kingoftheether.com/postmortem.html). The purpose of this challenge is to become the new king and avoid anyone else claiming their right to become owners. This level will help us understand that transactions to external contracts are not always successful and will require from us to write a contract with no `fallback` nor `receive` functions so it does not accept transfers from other users attempting to gain ownership:

```solidity {lineos=table,hl_lines=[17,18],lineofstart=1}
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

contract BecomeKing {
    address owner;

    constructor() public {
        owner = msg.sender;
    }

    function sendEther(address payable _dest) public payable {
        // Add ether when invoking a contract
        (bool sent, ) = _dest.call{value: address(this).balance}("");
        require(sent, "Failed to send ether!");
    }

    receive () external payable {
        revert('Thanks, but NOPE');
    }
}
```

You can read a walkthrough for this challenge [here](../ethernaut-9-king).