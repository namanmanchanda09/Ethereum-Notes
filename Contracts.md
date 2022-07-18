# Contracts

## 1. Visibility

**State variables**

- `public` variables differ from internal ones only in that the compiler automatically generates getter functions for them, which allows other contracts to read their values. When used within the same contract, the external access (e.g. `this.x`) invokes the getter while internal access (e.g. `x`) gets the variable value directly from storage. Setter functions are not generated so other contracts cannot directly modify their values.

- `internal` state variables can only be accessed from within the contract they are defined in and in derived contracts. They cannot be accessed externally. This is the default visibility level for state variables.

- `private` state variables are like internal ones but they are not visible in derived contracts.

**Function Visibility**

- `external` functions are part of the contract interface, which means they can be called from other contracts and via transactions. An external function `f` cannot be called internally (i.e. `f()` does not work, but `this.f()` works).

- `public` functions are part of the contract interface and can be either called internally or via message calls.

- `internal` functions can only be accessed from within the current contract or contracts deriving from it. They cannot be accessed externally. Since they are not exposed to the outside through the contract’s ABI, they can take parameters of internal types like mappings or storage references.

- `private` functions are like internal ones but they are not visible in derived contracts.

```solidity
contract A {
    uint private data = 42;
    string private name = "Alchemist";

    function getData()
    public
    view
    returns(uint) {
        return data;
    }

    function getName()
    internal
    view
    returns(string memory) {
        return name;
    }
}

contract B {

    function readData()
    public
    returns(uint) {
        A a = new A();
        return a.getData();
    }

}

contract C is A {

    function readName()
    public
    view
    returns(string memory) {
        return getName();
    }

}
```

## 2. Function Modifiers

Modifiers are inheritable properties of contracts and may be overridden by derived contracts, but only if they are marked `virtual`.

```solidity
contract Owned {
    address payable public owner;
    constructor() {
        owner = payable(msg.sender);
    }

    modifier onlyOwner {
        require(msg.sender == owner,
                "Only owner can call this fn");
        _;
    }
}

contract Destructible is Owned {
    function destroy() public onlyOwner {
        selfdestruct(owner); // deletes the owner of this contract
    }
}
```

Modifiers can receive arguments.

```solidity
contract A {
    modifier costs(uint price) {
        if(price<10){
            revert("price should be greater than 10");
        }
        _;
        }

    function fn(uint _price) public costs(_price) pure returns(bool){
        return true;
    }

}
```

**Reentrancy modifier**

This prevents `msg.sender` to call `f` again while a function call is already being executed.

```solidity
contract Mutex {
    bool internal locked;

    modifier noReentrancy() {
        require(
            !locked,
            "Reentrant call"
        );
        locked = true;
        _;
        locked = false;
    }

    function f()
    public
    noReentrancy
    returns(uint) {
        (bool success,) = msg.sender.call("");
        require(success);
        return 42;
    }
}
```
[more on reentrancy](https://www.youtube.com/watch?v=4Mm3BCyHtDY)

Multiple modifiers are applied to a function by specifying them in a whitespace-separated list and are evaluated in the order presented. Modifiers cannot implicitly access or change the arguments and return values of functions they modify. Their values can only be passed to them explicitly at the point of invocation. Explicit returns from a modifier or function body only leave the current modifier or function body. Return variables are assigned and control flow continues after the `_` in the preceding modifier.

```solidity
contract Mutex {
    uint public x;

    modifier check() {
        require(true);
        _;
        x = 42;
    }

    function f()
    public
    check {
        x = 10;
    }
}
```

After calling `f` the value of `x` would be `42`.

**NOTE**

*The `_` symbol can appear in the modifier multiple times. Each occurrence is replaced with the function body.*

Arbitrary expressions are allowed for modifier arguments and in this context, all symbols visible from the function are visible in the modifier. Symbols introduced in the modifier are not visible in the function (as they might change by overriding).

## 3. Constant And Immutable variables

For `constant` variables, the value has to be a constant at compile time and it has to be assigned where the variable is declared. Any expression that accesses storage, blockchain data (e.g. `block.timestamp`, `address(this).balance` or `block.number`) or execution data (`msg.value` or `gasleft()`) or makes calls to external contracts is disallowed.

Variables declared as `immutable` are a bit less restricted than those declared as `constant`: Immutable variables can be assigned an arbitrary value in the constructor of the contract or at the point of their declaration. They can be assigned only once and can, from that point on, be read even during construction time.

Both of the following would work.

```solidity
contract A {

    uint public immutable name=42;

}

contract B {
    uint public immutable name;
    constructor(){
        name=42;
    }
}
```

## 4. Functions

**Fn Parameters**

```solidity
contract A {
    uint[2][] public arr;
    function setArray(uint[2][] memory _arr) external {
        arr = _arr;
    }
}
```
 **Return Variables**
 
 ```solidity
 contract A {
    function fn(uint a, uint b) public pure returns(uint sum, uint product) {
        sum=a+b;
        product=a*b;
    }
}
```

**State Mutability**

Functions can be declared `view` in which case they promise not to modify the state.

The following statements are considered modifying the state:

- Writing to state variables.
- Emitting events.
- Creating other contracts.
- Using selfdestruct.
- Sending Ether via calls.
- Calling any function not marked view or pure.
- Using low-level calls.
- Using inline assembly that contains certain opcodes.

Functions can be declared `pure` in which case they promise not to read from or modify the state. In particular, it should be possible to evaluate a `pure` function at compile-time given only its inputs and `msg.data`, but without any knowledge of the current blockchain state. This means that reading from `immutable` variables can be a non-pure operation.

NOTE

`It is not possible to prevent functions from reading the state at the level of the EVM, it is only possible to prevent them from writing to the state (i.e. only view can be enforced at the EVM level, pure can not).`

## 5. Special Functions

**Receive Ether Fn**

A contract can have at most one `receive` function, declared using `receive() external payable { ... }` (without the `function` keyword). This function cannot have arguments, cannot return anything and must have `external` visibility and `payable` state mutability. It can be virtual, can override and can have modifiers.

```solidity
contract A {
    event Received(uint amount);
    receive() external payable{
        emit Received(msg.value);
    }

    function getBalance() public view returns(uint) {
        return address(this).balance;
    }
}

contract B {
    constructor() payable{}
    function sendEther(address payable _a, uint _amount) public {
        _a.transfer(_amount);
    } 
}
```

**Fallback Fn**

A contract can have at most one `fallback` function, declared using either `fallback () external [payable]` or `fallback (bytes calldata input) external [payable] returns (bytes memory output)` (both without the `function` keyword). This function must have `external` visibility. A fallback function can be virtual, can override and can have modifiers.

The fallback function is executed on a call to the contract if none of the other functions match the given function signature, or if no data was supplied at all and there is no receive Ether function. The fallback function always receives data, but in order to also receive Ether it must be marked `payable`.

```solidity
contract A {

    event ReceiveEmitted(uint amount);
    event FallbackEmitted(uint amount);

    receive() external payable{
        emit ReceiveEmitted(msg.value);
    }

    fallback() external payable{
        emit FallbackEmitted(msg.value);
    }

    function getBalance() public view returns(uint) {
        return address(this).balance;
    }
}

contract B {

    constructor() payable{}

    function sendEther(address payable _a) public {
        (bool success, ) = _a.call("nonExistingFunction()"); // fallback called
        require(success);
    }

    function sendEther2(address payable _a) public {
        (bool success, ) = _a.call{value: 1 ether}("nonExistingFunction()"); //fallback called
        require(success);
    }

    function sendEther3(address payable _a) public {
        (bool success, ) = _a.call{value: 1 ether}(""); // receive called
        require(success);
    }
}
```

`fallback` is called for all messages sent to a contract except plain Ether transfers. Any call with non-empty calldata will execute `fallback`. For plain Ether transfers, i.e for every call with empty calldata, `receive` is called.

## 6. Fn overloading

A contract can have multiple functions of the same name but with different parameter types. This process is called “overloading” and also applies to inherited functions.

```solidity
contract A {

    function fn(uint value) public pure returns(uint out) {
        out = value;
    }

    function fn(uint value, bool really) public pure returns(uint out) {
        if (really) {
            out = value;
        }
    }
}
```

## 7. Events

Solidity events give an abstraction on top of the EVM’s logging functionality. Applications can subscribe and listen to these events through the RPC interface of an Ethereum client.

Events are inheritable members of contracts. When you call them, they cause the arguments to be stored in the transaction’s log - a special data structure in the blockchain. These logs are associated with the address of the contract, are incorporated into the blockchain, and stay there as long as a block is accessible (forever as of now, but this might change with Serenity). The Log and its event data is not accessible from within contracts (not even from the contract that created them).

It is possible to request a Merkle proof for logs, so if an external entity supplies a contract with such a proof, it can check that the log actually exists inside the blockchain. You have to supply block headers because the contract can only see the last 256 block hashes.

You can add the attribute `indexed` to up to three parameters which adds them to a special data structure known as [“topics”](https://docs.soliditylang.org/en/v0.8.15/abi-spec.html#abi-events) instead of the data part of the log. A topic can only hold a single word (32 bytes) so if you use a reference type for an indexed argument, the Keccak-256 hash of the value is stored as a topic instead.

All parameters without the `indexed` attribute are ABI-encoded into the data part of the log.

Topics allow you to search for events, for example when filtering a sequence of blocks for certain events. You can also filter events by the address of the contract that emitted the event.

## 8. Inheritance

When a contract inherits from other contracts, only a single contract is created on the blockchain, and the code from all the base contracts is compiled into the created contract. This means that all internal calls to functions of base contracts also just use internal function calls (`super.f(..)` will use JUMP and not a message call).

State variable shadowing is considered as an error. A derived contract can only declare a state variable `x`, if there is no visible state variable with the same name in any of its bases.

**Constructor**

A constructor is an optional function that is executed upon contract creation.

```solidity
contract X {
    string public name;

    constructor(string memory _name) {
        name=_name;
    }
}


contract Y {
    string public text;

    constructor(string memory _text) {
        text=_text;
    }
}

// both are correct ways to initialize parent contracts

contract B is X("My name's X"), Y("My name's Y") {}

contract C is X, Y {
    constructor(string memory _name, string memory _text) X(_name) Y(_text) {}
}
```

Parent constructors are always called in the order of inheritance regardless of the order of parent contracts listed in the constructor of the child contract. 


```solidity
// Order of constructors called:
// 1. X
// 2. Y
// 3. D

contract D is X, Y {
    constructor() X("X was called") Y("Y was called") {}
}

contract E is X, Y {
    constructor() Y("Y was called") X("X was called") {}
}
```

If a derived contract does not specify the arguments to all of its base contracts’ constructors, it must be declared abstract.

```solidity
contract Base {
    uint x;
    constructor(uint x_) { x = x_; }
}

// Either directly specify in the inheritance list...
contract Derived1 is Base(7) {
    constructor() {}
}

// or through a "modifier" of the derived constructor...
contract Derived2 is Base {
    constructor(uint y) Base(y * y) {}
}

// or declare abstract...
abstract contract Derived3 is Base {
}

// and have the next concrete derived contract initialize it.
contract DerivedFromDerived is Derived3 {
    constructor() Base(10 + 10) {}
}
```


- Solidity supports multiple inheritance. Contracts can inherit other contract by using the `is` keyword.
- Function that is going to be overridden by a child contract must be declared as `virtual`.
- Function that is going to override a parent function must use the keyword `override`.
- Order of inheritance is important.
- You have to list the parent contracts in the order from *most base-like* to *most derived*.

**Linearization**

```solidity

/* Graph of inheritance
    A
   / \
  B   C
 / \ /
F  D,E 

*/


contract A {
    function fn() public pure virtual returns(string memory) {
        return "A"; // A
    }
}

contract B is A {
    function fn() public pure override virtual returns(string memory) {
        return "B"; // B
    }

}

contract C is A {
    function fn() public pure virtual override returns (string memory) {
        return "C"; // C
    }
}

contract D is B, C {
    function fn() public pure override(B,C) returns(string memory) {
        return super.fn(); // C
    }
}

contract E is C, B {
    function fn() public pure override(C, B) returns(string memory) {
        return super.fn(); // B
    }
}

contract F is A, B {
    function fn() public pure override(A, B) returns(string memory) {
        return super.fn(); // B
    }
}

contract G is B, A { // error
}
```

Contracts can inherit from multiple parent contracts. When a function is called that is defined multiple times in different contracts, parent contracts are searched from `right to left`, and `in depth-first manner`.

Inheritance must be ordered from `most base-like` to `most derived`. In contract `G` - we firstly try to inherit B and then A, but A is the parent contract of `B` i.e more base like than `B` hence defining `G` gives an error.

**Shadowing Inherited State Variables**

```solidity
contract A {
    string public name = "Contract A";

    function getName() public view returns(string memory) {
        return name;
    }
}

contract B is A {
    constructor() {
        name = "Contract B";
    }
}
```

- Functions with the `private` visibility cannot be `virtual`.
- Functions without implementation have to be marked `virtual` outside of interfaces. In interfaces, all functions are automatically considered `virtual`.
- Starting from Solidity *0.8.8*, the `override` keyword is not required when overriding an interface function, except for the case where the function is defined in multiple bases.

**Function Overriding**

Base functions can be overriden by inheriting contracts to change their behavior if they are marked as `virtual`. The overriding function must then use the `override` keyword in the function header. The overriding function may only change the visibility of the overridden function from `external` to `public`. The mutability may be changed to a more strict one following the order: `nonpayable` can be overridden by `view` and `pure`. `view` can be overridden by `pure`. `payable` is an exception and cannot be changed to any other mutability.

```solidity
contract Base
{
    function foo() virtual external view {}
}

contract Middle is Base {}

contract Inherited is Middle
{
    function foo() override public pure {}
}
```

For multiple inheritance, the most derived base contracts that define the same function must be specified explicitly after the `override` keyword.

```solidity
contract Base1
{
    function foo() virtual public {}
}

contract Base2
{
    function foo() virtual public {}
}

contract Inherited is Base1, Base2
{
    // Derives from multiple bases defining foo(), so we must explicitly
    // override it
    function foo() public override(Base1, Base2) {}
}
```

**Modifier Overriding**

There is no overloading for modifiers. 

```solidity
contract Base1
{
    modifier foo() virtual {_;}
}

contract Base2
{
    modifier foo() virtual {_;}
}

contract Inherited is Base1, Base2
{
    modifier foo() override(Base1, Base2) {_;}
}
```






