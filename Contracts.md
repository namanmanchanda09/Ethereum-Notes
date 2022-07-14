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








