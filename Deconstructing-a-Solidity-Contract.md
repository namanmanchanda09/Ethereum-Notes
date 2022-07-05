# Deconstructing a Solidity Contract

## 1. Introduction

**The contract**
```solidity
// SPDX-License-Identifier: GPL-3.0
pragma solidity ^0.4.24;

contract BasicToken {
  
  uint256 totalSupply_;
  mapping(address => uint256) balances;
  
  constructor(uint256 _initialSupply) public {
    totalSupply_ = _initialSupply;
    balances[msg.sender] = _initialSupply;
  }

  function totalSupply() public view returns (uint256) {
    return totalSupply_;
  }

  function transfer(address _to, uint256 _value) public returns (bool) {
    require(_to != address(0));
    require(_value <= balances[msg.sender]);
    balances[msg.sender] = balances[msg.sender] - _value;
    balances[_to] = balances[_to] + _value;
    return true;
  }

  function balanceOf(address _owner) public view returns (uint256) {
    return balances[_owner];
  }
}
```

Once the contract is compiled, we get the Bytecode & the ABI. Bytecode is what's necessary for the EVM.

**The ByteCode**
```
608060405234801561001057600080fd5b5060405160208061021783398101604090815290516000818155338152600160205291909120556101d1806100466000396000f3006080604052600436106100565763ffffffff7c010000000000000000000000000000000000000000000000000000000060003504166318160ddd811461005b57806370a0823114610082578063a9059cbb146100b0575b600080fd5b34801561006757600080fd5b506100706100f5565b60408051918252519081900360200190f35b34801561008e57600080fd5b5061007073ffffffffffffffffffffffffffffffffffffffff600435166100fb565b3480156100bc57600080fd5b506100e173ffffffffffffffffffffffffffffffffffffffff60043516602435610123565b604080519115158252519081900360200190f35b60005490565b73ffffffffffffffffffffffffffffffffffffffff1660009081526001602052604090205490565b600073ffffffffffffffffffffffffffffffffffffffff8316151561014757600080fd5b3360009081526001602052604090205482111561016357600080fd5b503360009081526001602081905260408083208054859003905573ffffffffffffffffffffffffffffffffffffffff85168352909120805483019055929150505600a165627a7a72305820a5d999f4459642872a29be93a490575d345e40fc91a7cccb2cf29c88bcdaf3be0029
```

Deploy the above contract using Remix & the following settings.
- Javascript VM
- `10000` as initial supply
- `version:0.4.24+commit.e67f0147.Emscripten.clang` as compiler version
- *Enable Optimization* must be selected

Once deployed, click on **Debug** transaction. This will activate *Debugger* tab in Remix. Take a look at *Instructions* section.
This is the disassembled bytecode of the contract. Disassembly sounds rather intimidating, but it’s quite simple, really. If you scan the raw bytecode by bytes (two characters at a time), the EVM identifies specific opcodes that it associates to particular actions. For example:

|  HEX |     OPCODE    |
| ---- | ------------  |
| 0x60 |      PUSH     |
| 0x01 |      ADD      |
| 0x02 |      MUL      |
| 0x00 |      STOP     |

 An opcode, in the end, can only push or consume items from the EVM’s stack, memory, or storage belonging to the contract. That’s it.
 Here's a list of all [opcodes](https://github.com/ethereum/pyethereum/blob/master/ethereum/opcodes.py).

**Instructions**

Each line in the disassembled code above is an instruction for the EVM to execute. Each instruction contains an opcode. For example, let’s take one of those instructions, instruction 88, which pushes the number 4 to the stack. This particular disassembler interprets instructions as follows:

```
88 PUSH1 0x04
|  |     |     
|  |     Hex value for push.
|  Opcode.
Instruction number.
```

**Deconstruction diagram**

More on this later.

<img src="https://lh5.googleusercontent.com/Jq2znqr5dabdpzYqmuyDsjq6HZzL6iTe2iWXqAJbuEzdMPyQJiRQZCqkJZ6qSo9GPjczGSIuvxxt3HMvonoOFlu-jRwqpfcraMsQ9NOAB3xCtg4f0Ww1oApGHO6Mr7GfTlboh862">

Here's the complete [diagram](https://lh5.googleusercontent.com/Jq2znqr5dabdpzYqmuyDsjq6HZzL6iTe2iWXqAJbuEzdMPyQJiRQZCqkJZ6qSo9GPjczGSIuvxxt3HMvonoOFlu-jRwqpfcraMsQ9NOAB3xCtg4f0Ww1oApGHO6Mr7GfTlboh862). Keep it handy.

## 2. Creation vs Runtime

We'll be going through each OPCODE and trying to make sense of how solidity handles it. For now, let's see `JUMP`, `JUMPI`, `JUMPDEST`, `RETURN`, and `STOP` & **ignore the rest**.

When the EVM executes code, it does so top down with no exceptions — i.e., there are no other entry points to the code. It always starts from the top. It can jump around, yes, and that’s exactly what `JUMP` and `JUMPI` do.
- `JUMP` takes the topmost value from the stack and moves execution to that location. The target location must contain a `JUMPDEST` opcode, or, otherwise execution fails.
- `JUMPDEST` marks a location as a valid jump target.
- `JUMPI` is same as `JUMP` but there must not be a **0** in the *2nd* position of the stack, otherwise, there will be no jump. So this is a conditional jump.
- `STOP` halts the execution of the contract.
- `RETURN` halts execution of the contract, but returns data from a portion of the **EVM's memory**.

*Back to remix debugger, take the slider and move all the way to left.*

The first few instructions can be ignored but we find our 1st `JUMPI` at instruction **11**. If it doesn’t jump, it will continue through instructions 12 to 15 and end up in a `REVERT`, which would halt execution. But if it does jump, it will skip these instructions to the location 16 (hex **0x0010**, which was pushed to the stack at instruction *8*. Instruction 16 is a `JUMPDEST`. So far so good.

At location *68* we find a `RETURN` opcode and a `STOP` at *69*. The control of this contract will always end at either *15* i.e `REVERT` or *68*. But the code ends at location *566*. So the set of instructions from (**0-69**) are called *creation code* of the contract. 

NOTE

`
The set of instructions we’ve just traversed (0 to 69) is what’s known as the “creation code” of a contract. It will never be a part of the contract’s code per se, but is only executed by the EVM once during the transaction that creates the contract. This piece of code is in charge of setting the created contract’s initial state, as well as returning a copy of its runtime code. The remaining 497 instructions (70 to 566) which, as we saw, will never be reached by the execution flow, are precisely the code that will be part of the deployed contract.
`

Take a look at the [deconstruction diagram](https://gists.rawgit.com/ajsantander/23c032ec7a722890feed94d93dff574a/raw/a453b28077e9669d5b51f2dc6d93b539a76834b8/BasicToken.svg) and see the first split we made b/w creation code and runtime code.

**Understanding Creation Code**

Now we will be splitting the creation code further to understand each part of it.

<img src="https://lh4.googleusercontent.com/H5-5mHxayCSR3zS1hgyy8UEC31y7n3d4ZOo6nf0-ZZ9Oz6idygz4o5_US6_MJejxDzTWg7bl9NOSiz_JuZIxmjH036Awhy2xD2RvLACuXHsqL06NV0goug8Z7O2F3ta7pDBlr-Le">

The creation code gets executed in a transaction, which returns a copy of the runtime code, which is the actual code of the contract. The contract’s constructor is part of the creation code; it will not be present in the contract’s code once it is deployed. Now we will be looking at all ~70 instructions in the debugger. Shift the slider to extreme left and use the down arrow to execute each opcode in Remix. Make sure to carefully observe the changes in stack, memory etc.

*FREE MEMORY POINTER*
```
000 PUSH1 80
002 PUSH1 40
004 MSTORE
```

In the above instructions - `PUSH` instructions are composed of 2 or more bytes. So `PUSH 80` is really two instructions. Hence constituting instruction #1 and #3.

- `PUSH1` pushes **1** byte onto the top of the stack.
- `MSTORE` grabs the last **2** items from the stack & stores one of them in memory.

```
mstore(0x40, 0x80)
         |     |
         |     What to store.
        Where to store.
(in memory)
```

The 1st opcode will push **0x80** to stack. The 2nd will push **0x40** to stack. See the stack changes for both the operations. The 3rd instruction will store the number `0x80` in the memory at position `0x40`. At this point, check the memory changes. 

*NON PAYABLE CHECK*
```
005 CALLVALUE
006 DUP1
007 ISZERO
008 PUSH2 0010
011 JUMPI
012 PUSH1 00
014 DUP1
015 REVERT
```

- `CALLVALUE` pushes the amount of wei involved in the creation transaction.
- `DUP1` duplicated the first element on the stack.
- `ISZERO` replaces the top element of stack by 1 if it's 0.
- `PUSH2` can push **2** bytes to the stack.

In solidity we can write the above chunk of assembly like

```solidity
if(msg.value != 0) revert();
```

This code was not actually part of our original Solidity source, but was instead injected by the compiler because we did not declare the constructor as payable. In the most recent versions of Solidity, functions that do not explicitly declare themselves as payable cannot receive ether.

Now the `JUMPI` at *11* will skip through instructions 12-15 and jump to *16* if there is no ether involved. Otherwise `REVERT` will execute with both parameters as 0. Make sure to follow everything along in Remix.

*RETRIEVE CONSTRUCTOR PARAMETERS*
```
016 JUMPDEST
017 POP
018 PUSH1 40
020 MLOAD
021 PUSH1 20
023 DUP1
024 PUSH2 0217
027 DUP4
028 CODECOPY
029 DUP2
030 ADD
031 PUSH1 40
033 SWAP1
034 DUP2
035 MSTORE
036 SWAP1
037 MLOAD
```

Continue running the following instructions in the Remix. Instruction *18* push `0x40` to stack. `MLOAD` replaces the first element of stack with memory data at `0x40`. That should be `0x80` pushed to the stack. Then `0x20` will be pushed and duplicated. Next `PUSH2 0217` pushes `0x0217` (decimal 535) to the stack and duplicate the 4th value in stack i.e `0x80`. 

- `CODECOPY` takes 3 arguments from the stack top.

STACK
```
0:0x0000000000000000000000000000000000000000000000000000000000000080
1:0x0000000000000000000000000000000000000000000000000000000000000217
2:0x0000000000000000000000000000000000000000000000000000000000000020
```

```
codecopy(0x80, 0x0217, 0x20)
         |     |        |
         |     |        number of bytes to copy
         |     instruction number to copy from.
         target position.
```

The entire code has 566 instructions. So `CODECOPY` is trying to copy last 32 bytes of code i.e from `0x0217`(decimal 535) to end of code. This is because when a contract with constructor arguments is deployed - the arguments are appended to the end of the code. Now check the memory at `0x80` and you'll see the `0x00..002710` i.e the number *10000* which we passed as initial supply.

The next few instructions are of utmost important. `DUP2` duplicates `0x80` and put on top of stack. Instruction 30 adds top two elements of the stack and replace with `0xa0` i.e `0x80` + `0x20`. Instruction 31 push `0x40` to stack, followed by swapping with the second element in stack and duplicating the 2nd element now. The stack at this instance looks like

STACK
```
0:0x0000000000000000000000000000000000000000000000000000000000000040
1:0x00000000000000000000000000000000000000000000000000000000000000a0
2:0x0000000000000000000000000000000000000000000000000000000000000040
3:0x0000000000000000000000000000000000000000000000000000000000000080
```

Now `MSTORE` would replace the memory at location `0x40` from `0x80` to `0xa0`. So this offset the memory value at `0x40` by 32 bytes. 

NOTE

`Solidity keeps track of something called a "free memory pointer": that is, a place in memory we can use to store stuff, with the guarantee that no one will overwrite it. So, since we stored the number 10000 in the old free memory position, we updated the free memory pointer by shifting it 32 bytes forward.`

So the **free memory pointer** or `mload(0x40, 0x80)` just means we will be writing to memory from this point on and keeping a record of the offset, each time we write a new entry. Every single function in Solidity, when compiled to EVM bytecode, will initialize this pointer. The memory b/w `0x40` and `0x80` is nothing but a portion that Solidity reserves for calculating hashes for mappings and other data types.

Continuing with the instructions, the `SWAP1` will swap the top with element at 2ns position. `MLOAD` reads from memory at position `0x80` and push that to stack. This basically pushes our constructor argument i.e `0x2710` (decimal 10000) to top of stack. This is a common pattern in EVM bytecode generated by Solidity: before a function’s body is executed, the function’s parameters are loaded into the stack (whenever possible), so that the upcoming code can consume them — and that’s exactly what’s going to happen next.

*CONSTRUCTOR BODY*
```
038 PUSH1 00      
040 DUP2            
041 DUP2         
042 SSTORE
043 CALLER
044 DUP2
045 MSTORE
046 PUSH1 01
048 PUSH1 20
050 MSTORE
051 SWAP2
052 SWAP1
053 SWAP2
054 SHA3
055 SSTORE
```

The above opcodes make up the constructor body.

```solidity
totalSupply_ = _initialSupply;
balances[msg.sender] =  _initialSupply;
```

The first few are basic push and duplicate. First, 0 is pushed to the stack, then the second item in the stack is duplicated (that’s our 10000 number), and then the number 0 is duplicated and pushed to the stack, which is the position slot in storage of `totalSupply_`. Now, `SSTORE` can consume the values and still keep 10000 lying around for future use:

```
sstore(0x00, 0x2710) 
        |     |
        |     What to store.
        Where to store.
(in storage)
```

As soon as `SSTORE` is executed check the Remix Storage for the changes. It would look something like this
```javascript
{
	"0x290decd9548b62a8d60345a988386fc84ba6bc95484008f6362f93160ef3e563": {
		"key": "0x0000000000000000000000000000000000000000000000000000000000000000",
		"value": "0x0000000000000000000000000000000000000000000000000000000000002710"
	}
}
```

And so it added `10000` to the global storage of the contract. The next set of instructions will store *10000* in the `balances` mapping for the key of `msg.sender`.




