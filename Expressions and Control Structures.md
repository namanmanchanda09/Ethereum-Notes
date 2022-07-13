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

### 1. Contract creating using salt

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




