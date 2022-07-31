# Contracts

Solidity is a statically typed language. The concept of `null` or `undefined` value does not exist in Solidity but newly created variables always have a default value.

## 1. Value Types

### Booleans

 `bool` - possible values are `true`/`false`
 
 All the logical operators work in the usual way.
 
 ### Integers
 
 `int`/`uint` - signed & unsigned integers of various sizes. Integers in Solidity are restricted to a certain range. 

The range is as follows
 ```solidity
contract PrimitiveTypes {
    /*
        uint8   ranges from 0 to 2 ** 8 - 1
        uint16  ranges from 0 to 2 ** 16 - 1
        ...
        uint256 ranges from 0 to 2 ** 256 - 1
    */

    uint8 public u8 = 42;
    uint8 public f8 = 255;
    uint8 public g8 = 256; // produce an error
    uint256 public u256 = 8923892639283623;

    /*
        int256 ranges from -2 ** 255 to 2 ** 255 - 1
        int128 ranges from -2 ** 127 to 2 ** 127 - 1
    */

    int8 public i8 = 127;
    int public i256 = -963287329863;
}
```

The max/min values of integers can be obtained using `type(X)` where `X` is the type of integer.
```solidity
contract PrimitiveTypes {
    int8 public maxi8 = type(int8).max;
    int8 public mini8 = type(int8).min;

    uint256 public maxu256 = type(uint256).max;
}
```

**Operations on Integers**

Shifts





