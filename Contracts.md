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


