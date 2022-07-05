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

The first few instructions can be ignored but we find our 1st `JUMPI` at instruction **11**.
