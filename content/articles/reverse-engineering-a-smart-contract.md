---
title: Reverse Engineering a Smart Contract
date: 2022-11-28
---
{{<figure src="../images/force-7.png" caption="Image courtesy of OpenZeppelin">}}
# Reverse Engineering a smart-contract

NTM: Write an introductory paragraph here

### Verified contracts on Etherscan
Everything in the blockchain is verifiable, consistent and can be publicly accessible. Every smart contract should have their source code published and verified in Etherscan, or any other blockchain used for its deployment.

For instance, using a testing smart contract deployed at address `0x4cD4AD48e22EdfC8a43B8cb26d0Ac9ceeB9E75f1` in the `Goerli` network, we could easily recover its source code by checking the **Contract** tab in Ethernaut:

{{<figure src="../images/sc-etherscan-1.png" caption="Figure 1 - Contract tab on Etherscan">}}

With this information we could now use our favorite IDE to inspect the contract's implementation and start our analysis.

### Bytecode decompilation
However, there may be scenarios where a smart contract does not have published or verified its source code, and we will be required to reverse engineer a contract by looking at its EVM bytecode. 

For example, lets assume the following EVM bytecode for one of the contracts that we have published in the Goerli network and there is no source code published:

```bash
0x608060405234801561001057600080fd5b506004361061007d5760003560e01c8063423f6cef1161005b578063423f6cef146100dc57806370a082311461010c57806395d89b411461013c578063b9c134771461015a5761007d565b806306fdde031461008257806318160ddd146100a0578063313ce567146100be575b600080fd5b61008a61018a565b60405161009791906105eb565b60405180910390f35b6100a861021c565b6040516100b5919061060d565b60405180910390f35b6100c6610226565b6040516100d39190610628565b60405180910390f35b6100f660048036038101906100f1919061052e565b61022f565b60405161010391906105d0565b60405180910390f35b61012660048036038101906101219190610505565b610321565b604051610133919061060d565b60405180910390f35b610144610369565b60405161015191906105eb565b60405180910390f35b610174600480360381019061016f919061052e565b6103fb565b60405161018191906105d0565b60405180910390f35b60606003805461019990610771565b80601f01602080910402602001604051908101604052809291908181526020018280546101c590610771565b80156102125780601f106101e757610100808354040283529160200191610212565b820191906000526020600020905b8154815290600101906020018083116101f557829003601f168201915b5050505050905090565b6000600254905090565b60006012905090565b6000806000803373ffffffffffffffffffffffffffffffffffffffff1673ffffffffffffffffffffffffffffffffffffffff168152602001908152602001600020549050828161027f91906106b5565b6000803373ffffffffffffffffffffffffffffffffffffffff1673ffffffffffffffffffffffffffffffffffffffff16815260200190815260200160002081905550826000808673ffffffffffffffffffffffffffffffffffffffff1673ffffffffffffffffffffffffffffffffffffffff168152602001908152602001600020600082825461030f919061065f565b92505081905550600191505092915050565b60008060008373ffffffffffffffffffffffffffffffffffffffff1673ffffffffffffffffffffffffffffffffffffffff168152602001908152602001600020549050919050565b60606004805461037890610771565b80601f01602080910402602001604051908101604052809291908181526020018280546103a490610771565b80156103f15780601f106103c6576101008083540402835291602001916103f1565b820191906000526020600020905b8154815290600101906020018083116103d457829003601f168201915b5050505050905090565b6000806000803373ffffffffffffffffffffffffffffffffffffffff1673ffffffffffffffffffffffffffffffffffffffff1681526020019081526020016000205490508281036000803373ffffffffffffffffffffffffffffffffffffffff1673ffffffffffffffffffffffffffffffffffffffff16815260200190815260200160002081905550826000808673ffffffffffffffffffffffffffffffffffffffff1673ffffffffffffffffffffffffffffffffffffffff16815260200190815260200160002060008282540192505081905550600191505092915050565b6000813590506104ea81610812565b92915050565b6000813590506104ff81610829565b92915050565b60006020828403121561051757600080fd5b6000610525848285016104db565b91505092915050565b6000806040838503121561054157600080fd5b600061054f858286016104db565b9250506020610560858286016104f0565b9150509250929050565b610573816106fb565b82525050565b600061058482610643565b61058e818561064e565b935061059e81856020860161073e565b6105a781610801565b840191505092915050565b6105bb81610727565b82525050565b6105ca81610731565b82525050565b60006020820190506105e5600083018461056a565b92915050565b600060208201905081810360008301526106058184610579565b905092915050565b600060208201905061062260008301846105b2565b92915050565b600060208201905061063d60008301846105c1565b92915050565b600081519050919050565b600082825260208201905092915050565b600061066a82610727565b915061067583610727565b9250827fffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff038211156106aa576106a96107a3565b5b828201905092915050565b60006106c082610727565b91506106cb83610727565b9250828210156106de576106dd6107a3565b5b828203905092915050565b60006106f482610707565b9050919050565b60008115159050919050565b600073ffffffffffffffffffffffffffffffffffffffff82169050919050565b6000819050919050565b600060ff82169050919050565b60005b8381101561075c578082015181840152602081019050610741565b8381111561076b576000848401525b50505050565b6000600282049050600182168061078957607f821691505b6020821081141561079d5761079c6107d2565b5b50919050565b7f4e487b7100000000000000000000000000000000000000000000000000000000600052601160045260246000fd5b7f4e487b7100000000000000000000000000000000000000000000000000000000600052602260045260246000fd5b6000601f19601f8301169050919050565b61081b816106e9565b811461082657600080fd5b50565b61083281610727565b811461083d57600080fd5b5056fea26469706673582212204ad696d4ca54f78e728c33a6cfa047d978e4d8e8bd34bc381af14bbe4148be1064736f6c63430008040033
```

If we map all the opcodes above into readable instructions, we will get the following code:

```bash
00: 6080 PUSH1 0x80 
02: 6040 PUSH1 0x40 
04: 52 MSTORE
...omitted for brevity...
```

From above code snippet, we can see 2 opcodes, `PUSH1` and `MSTORE`. The opcode `PUSH1` pushes 1-byte integer into the stack for future use (In the EVM, integers are range from 1-byte to 32-byte long, using `PUSH1`, `PUSH2`, ... , `PUSH32`, respectively, as opcodes). In this case it pushes `0x80` and `0x40` into the stack, then use the `MSTORE` opcode which takes the two items in the stack and performs a memory write operation (`MSTORE(0x40, 0x80)`). But let's just not focus on reversing opcodes for now.

Etherscan provides a functionality to represent EVM bytecode into their respective opcodes, by clicking on the **Switch To Opcodes View** button:

{{<figure src="../images/sc-etherscan-2.png" caption="Figure 3 - Smart contract bytecode view">}}

This will result in an output that should match with the EVM bytecode presented few lines above:

{{<figure src="../images/sc-etherscan-3.png" caption="Figure 3 - Smart contract opcodes view">}}

Additionally, you can use a EVM decompiler like [panoramix](https://github.com/palkeo/panoramix) (same one used by Etherscan and palkeo, among others) to decompile the EVM bytecode and obtain an approximation to the original smart contract source code:

```bash
root@vmi828562 ~# panoramix 0x608060405234801561001057600080fd5b506004361061007d5760003560e01c8063423f6cef1161005b578063423f6cef146100dc57806370a082311461010c57806395d89b411461013c578063b9c134771461015a5761007d565b806306fdde031461008257806318160ddd146100a0578063313ce567146100be575b600080fd5b61008a61018a565b60405161009791906105eb565b60405180910390f35b6100a861021c565b6040516100b5919061060d565b60405180910390f35b6100c6610226565b6040516100d39190610628565b60405180910390f35b6100f660048036038101906100f1919061052e565b61022f565b60405161010391906105d0565b60405180910390f35b61012660048036038101906101219190610505565b610321565b604051610133919061060d565b60405180910390f35b610144610369565b60405161015191906105eb565b60405180910390f35b610174600480360381019061016f919061052e565b6103fb565b60405161018191906105d0565b60405180910390f35b60606003805461019990610771565b80601f01602080910402602001604051908101604052809291908181526020018280546101c590610771565b80156102125780601f106101e757610100808354040283529160200191610212565b820191906000526020600020905b8154815290600101906020018083116101f557829003601f168201915b5050505050905090565b6000600254905090565b60006012905090565b6000806000803373ffffffffffffffffffffffffffffffffffffffff1673ffffffffffffffffffffffffffffffffffffffff168152602001908152602001600020549050828161027f91906106b5565b6000803373ffffffffffffffffffffffffffffffffffffffff1673ffffffffffffffffffffffffffffffffffffffff16815260200190815260200160002081905550826000808673ffffffffffffffffffffffffffffffffffffffff1673ffffffffffffffffffffffffffffffffffffffff168152602001908152602001600020600082825461030f919061065f565b92505081905550600191505092915050565b60008060008373ffffffffffffffffffffffffffffffffffffffff1673ffffffffffffffffffffffffffffffffffffffff168152602001908152602001600020549050919050565b60606004805461037890610771565b80601f01602080910402602001604051908101604052809291908181526020018280546103a490610771565b80156103f15780601f106103c6576101008083540402835291602001916103f1565b820191906000526020600020905b8154815290600101906020018083116103d457829003601f168201915b5050505050905090565b6000806000803373ffffffffffffffffffffffffffffffffffffffff1673ffffffffffffffffffffffffffffffffffffffff1681526020019081526020016000205490508281036000803373ffffffffffffffffffffffffffffffffffffffff1673ffffffffffffffffffffffffffffffffffffffff16815260200190815260200160002081905550826000808673ffffffffffffffffffffffffffffffffffffffff1673ffffffffffffffffffffffffffffffffffffffff16815260200190815260200160002060008282540192505081905550600191505092915050565b6000813590506104ea81610812565b92915050565b6000813590506104ff81610829565b92915050565b60006020828403121561051757600080fd5b6000610525848285016104db565b91505092915050565b6000806040838503121561054157600080fd5b600061054f858286016104db565b9250506020610560858286016104f0565b9150509250929050565b610573816106fb565b82525050565b600061058482610643565b61058e818561064e565b935061059e81856020860161073e565b6105a781610801565b840191505092915050565b6105bb81610727565b82525050565b6105ca81610731565b82525050565b60006020820190506105e5600083018461056a565b92915050565b600060208201905081810360008301526106058184610579565b905092915050565b600060208201905061062260008301846105b2565b92915050565b600060208201905061063d60008301846105c1565b92915050565b600081519050919050565b600082825260208201905092915050565b600061066a82610727565b915061067583610727565b9250827fffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff038211156106aa576106a96107a3565b5b828201905092915050565b60006106c082610727565b91506106cb83610727565b9250828210156106de576106dd6107a3565b5b828203905092915050565b60006106f482610707565b9050919050565b60008115159050919050565b600073ffffffffffffffffffffffffffffffffffffffff82169050919050565b6000819050919050565b600060ff82169050919050565b60005b8381101561075c578082015181840152602081019050610741565b8381111561076b576000848401525b50505050565b6000600282049050600182168061078957607f821691505b6020821081141561079d5761079c6107d2565b5b50919050565b7f4e487b7100000000000000000000000000000000000000000000000000000000600052601160045260246000fd5b7f4e487b7100000000000000000000000000000000000000000000000000000000600052602260045260246000fd5b6000601f19601f8301169050919050565b61081b816106e9565b811461082657600080fd5b50565b61083281610727565b811461083d57600080fd5b5056fea26469706673582212204ad696d4ca54f78e728c33a6cfa047d978e4d8e8bd34bc381af14bbe4148be1064736f6c63430008040033
```

Resulting in the following code

## Installing Ghidra
The purpose of this section is to go over the installation and configuration of Ghidra and the `ghidra-evm` plugin. Which will help us throughout the reverse engineering process.

- Install [Ghidra 9.1.2](https://github.com/NationalSecurityAgency/ghidra/releases/tag/Ghidra_9.1.2_build) 
- Install OpenJDK-11: `sudo apt install openjdk-11-jdk`
- Install the `ghidra-bridge` plugin: `python3 -m pip install ghidra-bridge`
- Install the `ghidra-bridge` server: `python3 -m ghidra_bridge.install_server ~/ghidra_scripts` 
- Install the `evm-cfg-builder` tool used to extract a control flow graph (CFG) from EVM bytecode: `python3 -m pip install evm-cfg-builder`
- Clone the [Ghidra EVM module](https://github.com/adelapie/ghidra-evm) for reverse engineering smart contracts: `git clone https://github.com/adelapie/ghidra-evm.git ~/ghidra-evm`

The next step will be to install the Ghidra EVM Module, by opening Ghidra, and then accessing `File` > `Install Extensions...`

```bash 
python3 ~/ghidra-evm/helper/evm_helper.py ~/smart_contract_security/int_overflow_ghidra/compiled_contract.evm

       _     _     _
  __ _| |__ (_) __| |_ __ __ _        _____   ___ __ ___
 / _` | '_ \| |/ _` | '__/ _` |_____ / _ \ \ / / '_ ` _ \
| (_| | | | | | (_| | | | (_| |_____|  __/\ V /| | | | | |
 \__, |_| |_|_|\__,_|_|  \__,_|      \___| \_/ |_| |_| |_| v.0.1
 |___/

[*] Parsing bytecode...
b'608060405234801561001057600080fd5b506004361061007d5760003560e01c8063423f6cef1161005b578063423f6cef146100dc57806370a082311461010c57806395d89b411461013c578063b9c134771461015a576100
...omitted for brevity...
[*] Setting analysis options....
[*] Creating CFG...
[*] Resolving jumps...
<cfg BasicBlock@795-79b>
Finishes at:  0x79b
	JUMP to: 0x7d2
<cfg BasicBlock@10-19>
Finishes at:  0x19
	JUMPI to:  0x7d
<cfg BasicBlock@a0-a7>
Finishes at:  0xa7
	JUMP to: 0x21c
```

{{<figure src="../images/sc-ghidra-1.png" caption="Figure 4 - Enabling Ghidra Bridge">}}

{{<figure src="../images/sc-ghidra-2.png" caption="Figure 5 - Opcodes parsed correctly">}}

{{<figure src="../images/sc-ghidra-3.png" caption="Figure 5 - Control flow graph (CFG)">}}

## Reverse Engineering process
unctions Signatures (4byte identifiers) / Functions name & arguments type recovery
-- Complete this, services that maintain a databse where you can provide a 4bytes signature and get the function's body.
https://www.4byte.directory/

`explorer.web3_sha3('0x'+'FUNCTION'.encode("utf-8").hex())`

- `0x423f6cef` - `safeTransfer(address, uint256)`
- `0x70a08231` - `balanceOf(address)`
- `0x95d89b41` - `symbol()`
- `0xb9c13477` - No signature available, although this one belongs to `unsafeTransfer(address, uint256)`
- `0x06fdde03` - `name()`
- `0x18160ddd` - `totalSupply()`
- `0x313ce567` - `decimals()`

Reason why there could be multiple matches for a signature is due to a hash collision, as we are just using the first 4 bytes.
Probably best approach is tu use panoramix

### Diagram
- sol2umc / standalon + etherscan
- 

### Dispatcher function
When you perform a transaction to a smart contract, the first piece of code your transaction will interact with is the contract's dispatcher. The dispatcher takes your transaction data and determines which function are you trying to interact with.

The first thing we will notice with the dispatcher function in this contract is a check to determine if `msg.value` is zero, if so, we will revert the transaction:

{{<figure src="../images/sc-re-1.png" caption="Figure 6 - CFG for dispatcher function">}}

> This behavior corresponds to an optimization enforced in the EVM. The compile in Solidity checks if any of the functions in the smart contract are payable, adding the check at the top of the `_dispatcher` method if no functions are found, rather than adding this check to each individual function.
> 
> Including the `msg.value` check at the top of the `_dispatcher` method helps to save gas, as the check is only performed once, rather than on each individual function.

Assuming `msg.value` is zero, the next basic block to be executed will compare if `CALLDATASIZE` (`msg.data.size`)is less than `0x4`. If so, it will jump to `0x7d`, which is the `_fallback()` function that will `REVERT` the contract. This is another common behavior enforced by the EVM, as Solidity uses the first 4 bytes of the call data to select which function needs to be executed, and if no function is specified the fallback method will be called, as shown below:

{{<figure src="../images/sc-re-2.png" caption="Figure 7 - Fallback implementation">}}

Moving forward within the `_dispatcher()` implementation, the next thing we observe is that our basic block branches off into two different paths:

{{<figure src="../images/sc-re-3.png" caption="Figure 8 - Dispatcher implementation">}}

This is another optimization performed by the EVM, which instead of performing a linear function search, it splits the execution into two branches, and depending on the function selector provided it will perform the search in one branch or another.

This logic starts its implementation on address `0x21`, where the `PUSH4` opcode pushes 4 bytes (`0x423f6cef`) into the stack and then performs a comparison using the `GT` opcode. If the function selector provided by the user is lower than `0x423f6cef`, the execution flow will continue on address `0x5b`, otherwise it will continue on address `0x2b`.

Lets assume our selector hash is bigger than `0x423f6cef`, that means we will jump into the following basic block:

{{<figure src="../images/sc-re-4.png" caption="Figure 9 - Selector jump">}}

This is where the real logic behind the `_dispatcher` method is implemented, the `DUP1` opcode duplicates the first item in the stack, which is the hash of the function selector that we submitted to the contract (First 4 bytes of `msg.data`). The next instruction is `PUSH4 0x423f6cef`, which pushes those 4 bytes to the top of the stack. Then, it performs a comparison with the two items off the stack by using the `EQ` opcode (the first item will be `0x423f6cef`, and the second item will be the function selector), and pushing the result into the stack. Then it pushes `0xdc` into the stack by using `PUSH2 0xdc` (`safeTransfer(address, uint256`), and last but not least, it will use the opcode `JUMPI` to jump into the previous address, if the condition evalued by the `EQ` opcode was `true`. Otherwise it will continue execution at address `0x36`.

-- Check this and write a better transition

The purpose of this contract is to exploit an integer underflow issue in the `unsafeTransfer` (`0xb9c13477`) function, but there is a peculiarity with this contract, as it has been compiled using Solidity 0.8.0.  In previous versions of Solidity (prior Solidity 0.8.0) an integer would automatically roll-over to a lower or higher number. 

However, the `unsafeTransfer` method uses in its implementation the `unchecked` keyword, which allows developers write more 
 efficient programs. By default, smart contracts using Solidity 0.8.0 or higher implements a series of opcodes that prior to performing the actual arithmetic, check for under/overflow and revert the transaction if it is detected.

 {{<figure src="../images/sc-re-5.png" caption="Figure 10 - unsafeTransfer and safeTransfer methods' implementation">}}

### Spotting the issue
**NTM**: Write an introductory paragraph explaining how to use Remix
**NTM**: Why using remix and its debugger is benefitial for the successful exploitation of this vulnerability

#### Inspecting the safeTransfer method

**NTM:** Move this initial paragraph and modify the introduction to this section.

To identify the vulnerability lets compile and deploy the vulnerable smart contract with Remix, using `1000` tokens as initial supply. Then let's submit a `tx` to `safeTransfer` providing any address and `10000` as amount. The `tx` will automatically revert providing an error message, as shown below:

```
  
[vm]
from: 0x5B3...eddC4
to: MyToken.safeTransfer(address,uint256) 0xd91...39138
value: 0 wei
data: 0x423...02710
logs: 0
hash: 0xc48...5b4db
Debug
transact to MyToken.safeTransfer errored: VM error: revert.
revert
	The transaction has been reverted to the initial state.
Note: The called function should be payable if you send value and the value you send should be less than your current balance.
Debug the transaction to get more information.
```

**NTM:** Add more information here

Debugging a `tx` in Remix can be tedious but necessary, as we will be able to step into and step back on each opcode, getting a better understanding on what's going on at a low level. But for now, let's just focus on the following opcodes and the stack values:

{{<figure src="../images/sc-iu-safetransfer.png" caption="Figure 11 - safeTransfer opcodes step by step">}}

This block  implements the logic that checks if `senderBalance` is less than `_value` by:
1. `SWAP3` swaps the top of the stack with the 4th last element (In this case the stack will remain the same).
2. `POP` pops a uint256 off the stack and discards it.
3. `DUP3` clones the third last value on the stack. As this operation is executed twice, it will add `0x0000000000000000000000000000000000000000000000000000000000002710` first, and then `0x00000000000000000000000000000000000000000000000000000000000003e8`, therefore the stack will look like this:
4. `LT` (lower than, `a < b`) performs a `uint256` comparison between the last two elements in the stack. In this case it will compare  `0x00000000000000000000000000000000000000000000000000000000000003e8 < 0x0000000000000000000000000000000000000000000000000000000000002710`, returning `1`.
5. `ISZERO` checks that the last element in the stack is `0` and pushes the result into the stack:
6. `PUSH2 06e6` pushes a 2-byte value (`0x06e6`) into the stack:
7. `JUMPI` conditional jump if condition is truthy, which in this case it is not
8. `PUSH2 06e5` and `PUSH 07ab` which will add new elements into the stack:
9. `JUMP` tells the EVM to have the instruction pointer jump to another location in the bytecode, by getting the location off the first value from the stack. It is worth mentioning that only offsets where the `JUMPDEST` opcode is located are considered valid jumping targets.

This last opcode will move the execution flow to the address `0x7ab`, causing the transaction to revert with return data (`REVERT`):

```bash
1963 JUMPDEST
1964 PUSH32 4e487b7100000000000000000000000000000000000000000000000000000000
1997 PUSH1 00
1999 MSTORE
2000 PUSH1 11
2002 PUSH1 04
2004 MSTORE
2005 PUSH1 24
2007 PUSH1 00
2009 REVERT
```


This is caused because the value submitted in our `tx` is larger than the funds available in the contract, which may cause an integer underflow.

#### Inspecting the unsafeTransfer method
As we did in the previous step, this time we will submit a `tx` to the `UnsafeTransfer` method and focus on the `SUB` opcode and how it  handles the result, using the debugger available in Remix for that.

To speed up the process, the following opcodes are a snapshot of what occurs when executing `_balances[msg.sender] = senderBalance - value` inside an `unchecked` block:

{{<figure src="../images/sc-iu-unsafetransfer.png" caption="Figure 12 - unsafeTransfer opcodes step by step">}}

1) The balance of the address (`0xAb8...35cb2`) that performs the `tx` has a total balance of `0`. The first last value in the stack is `0x2710` (`10000`) which corresponds to the funds currently available in the smart contract. The second last value in the stack contains our accont's balance, which is `0`.
2) The opcode `DUP2` clones the second last value on the stack (`0x0`).
3) `SUB` performs a uint256 substraction modulo using the first and second last values on the stack (`0x0 - 0x2710`). This is where the integer underflow occurs.
4) The result of the previous opcode returns `0xffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffd8f0`, this is because we have attempted to make a `uint256` negative, triggering the underflow and causing it to contain a high value instead.
5) The following opcodes corresponding to the instruction `_balances[_to] += _value;` will just set the new value as our current balance.

### Exploiting the smart contract
So you have made it to this point and you may wonder how can you exploit the vulnerability that you just spotted. To achieve this goal we will use hardhat, an Ethereum development environment. Explaining how to setup the environment and how it does work is beyond this article, but I strongly recommend you to visit the following resources:

- [Hardhat Documentation](https://hardhat.org/docs)
- [Hardhat Tutorial](https://hardhat.org/tutorial)

Assuming you already have a working environment, we will divide our exploit in two separate files, `script.js`