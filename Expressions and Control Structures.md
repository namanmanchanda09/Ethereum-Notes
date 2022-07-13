# Expressions and Control Structures

Parentheses can not be omitted for conditionals, but curly braces can be omitted around single-statement bodies. Note that there is no type conversion from non-boolean to boolean types as there is in C and JavaScript, so `if (1) { ... }` is not valid Solidity.

### 1. Fn calls
**Internal fn calls**
```solidity
contract Alchemist {
    function gm(uint _x) public pure returns(uint) {
        return fn() + _x;
    }

    function fn() public pure returns(uint) {
        return 42;
    }
}
```
These function calls are translated into simple jumps inside the EVM. This has the effect that the current memory is not cleared, i.e. passing memory references to internally-called functions is very efficient. Only functions of the same contract instance can be called internally. You should still avoid excessive recursion, as every internal function call uses up at least one stack slot and there are only 1024 slots available.

**External fn calls**

Functions can also be called using `this.fn()` and `alchemist.fn()` where `alchemist` is an instance of contract `Alchemist`. Calling the fn this way results in a message call and is not being called via jumps. Functions of other contracts have to be called externally. For an external call, all function arguments have to be copied to memory.

`A function call from one contract to another does not create its own transaction, it is a message call as part of the overall transaction.`

```solidity
contract Kafka {
    uint public x;
    function giveMeMoney(uint _a) public payable returns(uint) {
        x = _a;
        return x;
    }
}

contract Alchemist {
    receive() external payable{}
    
    function giveaway(Kafka kafka, uint _b) public payable returns(uint) {
        return kafka.giveMeMoney{value: msg.value}(_b);
    }
}
```

When calling functions of other contracts, you can specify the amount of Wei or gas sent with the call with the special options `{value: 10, gas: 10000}`.

**Named calls**

```solidity
contract Alchemist {

    string public name;
    uint public no;

    function fn(string memory _name, uint _no) private {
        name = _name;
        no = _no;
    }

    function callFn(string memory _nm, uint _n) public {
        fn({_no:_n, _name:_nm});
    }

}
```

The names of unused parameters can be omitted.

```solidity
contract Alchemist {

    string public name;

    function fn(string memory _name, uint) public returns(string memory) {
        name = _name;
        return name;
    }

}
```

### 2. Contract creating using salt

Contracts can be created using `new` keyword. The full code of the contract being created has to be known when the creating contract is compiled so recursive creation-dependencies are not possible.

When creating a contract, the address of the contract is computed from the address of the creating contract and a counter that is increased with each contract creation.

If you specify the option `salt` (a bytes32 value), then contract creation will use a different mechanism to come up with the address of the new contract:

It will compute the address from the address of the creating contract, the given salt value, the (creation) bytecode of the created contract and the constructor arguments.

In particular, the counter (“nonce”) is not used. This allows for more flexibility in creating contracts: You are able to derive the address of the new contract before it is created. Furthermore, you can rely on this address also in case the creating contracts creates other contracts in the meantime.

```solidity
contract A {
    uint public x;

    constructor(uint _x) {
        x = _x;
    }
}

contract Factory {

    function create(bytes32 salt, uint _a)
    public
    returns(address) {
        address predictedAddress = address(
            uint160(
                uint(
                    keccak256(
                        abi.encodePacked(
                            bytes1(0xff),
                            address(this),
                            salt,
                            keccak256(
                                abi.encodePacked(
                                    type(A).creationCode,
                                    abi.encode(_a)
                                )
                            )
                        )
                    )
                )
            )
        );

        A a = new A{salt:salt}(_a);
        require(address(a) == predictedAddress);
        return address(a);
    }
}
```

### 3. Assignments

Solidity internally allows tuples.

```solidity
contract Alchemist {
    function fn() public pure returns(uint, bool, string memory) {
        return (42, true, "per aspera ad astra");
    }

    function gm() public pure returns(string memory) {
        (,,string memory name)=fn();
        return name;
    }
}
```

**Complications for Arrays and Structs**

In following example, `g(x)` doesn't modify the state because it creates a copy in the memory. However `h(x)` modifies `x`.

```solidity
contract Alchemist {
    uint[20] public x;

    function f() public {
        g(x);
        h(x);
    }

    function g(uint[20] memory y) internal pure {
        y[2] = 3;
    }

    function h(uint[20] storage y) internal {
        y[3] = 4;
    }
}
```

### 4. Scopes

This is valid code but throws a warning.

```solidity
contract Alchemist {
    function fn() public pure returns(uint){
        uint x = 1;
        {
            x = 2;
            uint x;
        }
        return x; // 2
    }
}
```

### 5. Unchecked Arithmetic

Code inside `unchecked` isn't checked for mathematical over/underflows. Hence it's a way to save some gas. However, that means you could cause a critical error or leave a security loop that someone might exploit.

```solidity
contract Alchemist {

    function f(uint a, uint b) public pure returns(uint) {
        unchecked{
            return a-b; // 2**256-1
        }
    }

    function g(uint a, uint b) public pure returns(uint) {
        return a-b; // revert
    }
}
```

### 6. Error handling: Assert, Require, Revert and Exceptions

Solidity uses state-reverting exceptions to handle errors. Such an exception undoes all changes made to the state in the current call (and all its sub-calls) and flags an error to the caller.

When exceptions happen in a sub-call, they “bubble up” (i.e., exceptions are rethrown) automatically unless they are caught in a `try/catch` statement. Exceptions to this rule are send and the low-level functions `call`, `delegatecall` and `staticcall`: they return `false` as their first return value in case of an exception instead of “bubbling up”.

*Warning*

**The low-level functions `call`, `delegatecall` and `staticcall` return true as their first return value if the account called is non-existent, as part of the design of the EVM. Account existence must be checked prior to calling if needed.**

`Error` is used for “regular” error conditions while `Panic` is used for errors that should not be present in bug-free code.

**Panic via `assert` and Error via `require`**

Assert should only be used to test for internal errors, and to check invariants. A Panic exception is generated in the following situations. The error code supplied with the error data indicates the kind of panic.

1. 0x00: Used for generic compiler inserted panics.
2. 0x01: If you call `assert` with an argument that evaluates to false.
3. 0x11: If an arithmetic operation results in underflow or overflow outside of an `unchecked { ... }` block.
4. 0x12; If you divide or modulo by zero (e.g. `5 / 0` or `23 % 0`).
5. 0x21: If you convert a value that is too big or negative into an enum type.
6. 0x22: If you access a storage byte array that is incorrectly encoded.
7. 0x31: If you call `.pop()` on an empty array.
8. 0x32: If you access an array, `bytesN` or an array slice at an out-of-bounds or negative index (i.e. `x[i]` where `i >= x.length` or `i < 0`).
9. 0x41: If you allocate too much memory or create an array that is too large.
10. 0x51: If you call a zero-initialized variable of internal function type.


An `Error(string)` exception (or an exception without data) is generated by the compiler in the following situations:

1. Calling `require(x)` where `x` evaluates to `false`.
2. If you use revert() or revert("description").
3. If you perform an external function call targeting a contract that contains no code.
4. If your contract receives Ether via a public function without payable modifier (including the constructor and the fallback function).
5. If your contract receives Ether via a public getter function.

Internally, Solidity performs a revert operation (instruction `0xfd`). This causes the EVM to revert all changes made to the state. The reason for reverting is that there is no safe way to continue execution, because an expected effect did not occur. Because we want to keep the atomicity of transactions, the safest action is to revert all changes and make the whole transaction (or at least call) without effect.

A direct `revert` can be triggered as well.

```solidity
contract Alchemist {

    function f(uint a) public pure {
        if(a < 3) {
            revert("you stop writing solidity if you can only revert");
        }
    }
}
```

**Custom Errors**

Using a custom error instance will usually be much cheaper than a string description, because you can use the name of the error to describe it, which is encoded in only four bytes. A longer description can be supplied via NatSpec which does not incur any costs.

```solidity
contract Alchemist {

    error AnError(string errorName, uint no);

    function f(uint a) public pure {
        if(a < 3) {
            revert AnError("you stop writing solidity if you can only revert", a);
        }
    }
}
```

**`try`/`catch`**

to be added.





