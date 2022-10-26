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

### 0x4 Telephone
The idea behind this challnge is to call the `changeOwner` function from a smart contract. So `msg.sender` will be the helper contract address while `tx.origin` will always refers to the address who started the original transaction. Once `tx.origin != msg.sender` is true, you will become the contract's new owner.

You can read a walkthrough for this challenge [here](../ethernaut-4-telephone).

### 0x5 Token
The goal in this contract is to increase our balance. Despite there is a requirement that checks for potential overflows (`require(balances[msg.sender] - _vaalue >= 0);`), we are dealing with unsigned integers which are known to be prone to overflow and underflow issues. To pass this challenge we will have to call the `transfer` method with an amount of tokens greater than 20.

You can read a walkthrough for this challenge [here](../ethernaut-5-token).

### 0x6 Delegation
This challenge consists of a delegation contract that forwards any calls using `delegatecall` in its `fallback` function to a `Delegate` contract. To complete this challenge we must get familiar with the `delegatecall` instruction and understand that the call performed heere is executed in the caller's context, thus the `Delegate` contract will access the `Delegation` owner storage variable and `msg.sender`. To pass this challenge you simply need to call the `pwn()` function in th `Delegation` contract, despite it doesn't exist.

You can read a walkthrough for this challenge [here](../ethernaut-6-delegation).

### 0x7 Forcee
The purpose of this challenge is to send ether to the contract. Sending ether to a contract requires a `fallback` function to be implemented, however, one can force-send ether by calling the `selfdestruct` instruction on a contract containing ether.

You can read a walkthrough for this challenge [here](../ethernaut-7-force).