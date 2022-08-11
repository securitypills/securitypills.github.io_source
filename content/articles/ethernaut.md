---
title: Ethernaut Challenges
date: 2022-08-03
---
### Please read
 This is a work in progress article that will receive updates as we continue publishing detailed walkthroughs for each level.

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

You can read a walkthrough for this challenge [here](../ethernaut-2-fallout).

### 0x3 Coinflip
The goal in this challenge is to guess the coin flip ten times in a row. The contract uses the previous blockhash as the flip outcome. This challenge can be solved by creating a similar contract that mimics the same coin flipping logic and calls Ethernaut's original contract with the result obtained.

You can read a walkthrough for this challenge [here](../ethernaut-3-coinflip).