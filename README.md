# Learn Yul

## This repository is a collection of my notes taken in the process of learning Yul.

Still a work in progress.

## Description

Yul is an intermediate language that can be compiled to bytecode for different backends. It can be used in stand-alone mode and for “inline assembly” inside Solidity. The compiler uses Yul as an intermediate language in the IR-based code generator (“new codegen” or “IR-based codegen”). Yul is a good target for high-level optimisation stages that can benefit all target platforms equally.

# Yul Syntax

-   To write Yul code in Solidity, use the `assembly` keyword.

```solidity
contract C {
    function f() public {
        assembly {
            // Yul code goes here
        }
    }
}
```

-   To declarea a variable, use the `let` keyword.

```solidity
let x := 1
```

-

# Yul Types

### Yul has only 1 type: `bytes32`. This can hold any value. The compiler will automatically insert conversions as needed.

For example, the following function will return `true`, `10`, and `0x48656c6c6f20576f726c64210000000000000000000000000000000000000000` which is the bytes32 representation of the "Hello World!" string.

```solidity
function f() public pure returns (bool, uint256, bytes32) {
        bool x;
        uint256 y;
        bytes32 z;

        assembly {
            x := 1
            y := 0xa
            z := "Hello World!"
        }

        return( x, y, z);
    }
```

# Yul Basic Operations

### Yul has no overflow protection!

-   add(x, y) - addition
-   sub(x, y) - subtraction
-   mul(x, y) - multiplication
-   div(x, y) - division
-   mod(x, y) - modulo

For multiple operations, the innermost operation is executed first:

```solidity
let x := add(1, mul(2, 3)) // = add(1, 6) = 7
```

# For Loop

Both of the following examples are valid:

```solidity
    function forLoop(uint256 n) public {
        assembly {
            for {let i := 0} lt(i, n) {i := add(i, 1)} {
                // do something
            }
        }
    }

    function forLoop(uint256 n) public {
        assembly {
            let i := 0
            for { } lt(i, n) { } {
                // do something
                i := add(i, 1)
            }
        }
    }
```

# If Statement

Yul has no boolean type. Instead, any value other than `0` is considered true.

```solidity
    function ifTrue(uint256 n) public {
        assembly {
            if 2 { // 2 is true
                // do something
            }
        }
    }

    function ifFalse(uint256 n) public {
        assembly {
            if 0 { // 0 is false
                // do something
            }
        }
    }

    function negation(uint256 n) public {
        assembly {
            // if 0 is 0 result in true, negation of false
            if iszero(0) {
                // if true, do something
            }
        }
    }
```

# Storage Slots & Variables

## Single variables being stored in one slot:

-   Storage slots are 256-bit words. To get the storage slot of a variable, use the `.slot` keyword.
-   To load a value from storage, use the `sload` keyword and pass in the storage slot as a parameter.
-   To store a value to storage, use the `sstore` keyword and pass in the storage slot and value as parameters.

Example of setter and getter functions for a storage variable:

```solidity
    uint256 x;

    function set(uint256 _x) public {
        assembly {
            sstore(x.slot, _x)
        }
    }

    function get() public view returns (uint256 x_) {
        assembly {
            x_ := sload(x.slot)
        }
    }
```

## Multiple variables being packed into one slot:

-   Offset is the number of bytes from the start of the slot that the variable starts at. To get the offset of a variable, use the `.offset` keyword.

Example of setter and getter function for packed storage variables:

```solidity
    uint128 a;
    uint96 b;
    uint16 c;
    uint8 d;

    function set(uint16 _c) public {
        assembly{
            // Get the storage slot of the variable
            let wholeSlot := sload(c.slot)

            // Clear the variable's bits in the slot. Since it is a uint16, it is 2 bytes long.
            let cleared := and(wholeSlot, 0xffff0000ffffffffffffffffffffffffffffffffffffffffffffffffffffffff)

            // Shift the new value to the left by the offset of the variable multiplied by 8(1 byte = 8 bits)
            let shifted := shl(mul(c.offset, 8), _c)

            // Combine the cleared slot and the shifted value
            let newValue := or(shifted, cleared)

            // Store the new value in the slot
            sstore(c.slot, newValue)
        }
    }

    function get() public view returns (uint16 c_) {
        assembly {
            // Get the storage slot of the variable
            let wholeSlot := sload(c.slot)

            // Shift the slot to the right by the offset of the variable
            let shifted := shr(mul(c.offset, 8), wholeSlot)

            // Mask the slot to get the value of the variable
            c_ := and(shifted, 0xffff)
        }
    }
```

## Storage Arrays

### Fixed Arrays

-   To get the bytes32 value at a specific index of a fixed array, use the `sload` keyword and pass in the storage `slot of the array + index` as parameters.

```solidity
    uint256[5] arr;

    function get(uint256 index) public view returns (uint256 value) {
        assembly {
            value := sload(add(arr.slot, index))
        }
    }
```

-   For arrays of variables smaller than 32 bytes, the compiler will pack the variables into a single slot when possible.

```solidity
    uint128[4] arr;

    function getIndex1() public view returns (uint128 value) {
        bytes32 packed;
        assembly {
            // Get the first bytes32 of the array
            packed := sload(arr.slot)
            // Shift the bytes32 to the right by 16 bytes(128 bits) to get the value of the first variable
            value := shr(mul(16, 8), packed)
        }
    }
```

### Dynamic Arrays

-   To get the bytes32 value at a specific index of a dynamic array, use the `sload` keyword and pass in the `keccak256 of the  storage slot of the array + index` as parameters.

```solidity
    uint256[] arr;

    function get(uint256 index) public view returns (uint256 value) {
        uint256 slot;
        assembly {
            slot := arr.slot
        }
        bytes32 location = keccak256(abi.encode(slot));

        assembly{
            value := sload(add(location, index))
        }
    }
```

# Mappings

## Simple Mapping

Mappings behave similar to arrays, but it concatenates the key and the mapping's storage slot to get the location of the value.

```solidity
    mapping(uint256 => uint256) map;

    function get(uint256 key) public view returns (uint256 value) {
        bytes32 slot;
        assembly {
            slot := map.slot
        }
        bytes32 location = keccak256(abi.encode(key, uint256(slot)));

        assembly{
            value := sload(location)
        }
    }
```

## Nested Mappings

Nested mappings are similar, but use hashes of hashes to get the location of the value. The concatenation and the hashing is done from right to left.

```solidity
    mapping(uint256 => mapping(uint256 => uint256)) map;

    function get(uint256 key1, uint256 key2) public view returns (uint256 value) {
        bytes32 slot;
        assembly {
            slot := map.slot
        }
        bytes32 location = keccak256(abi.encode(key2, keccak256(abi.encode(key1, uint256(slot)))));

        assembly{
            value := sload(location)
        }
    }
```

## Mapping of Arrays

```solidity
    mapping(address => uint256[]) map;

    function get(address key, uint256 index) public view returns (uint256 value) {
        bytes32 slot;
        assembly {
            slot := map.slot
        }
        bytes32 location = keccak256(abi.encode(keccak256(abi.encode(key, uint256(slot)))));

        assembly{
            value := sload(add(location, index))
        }
    }
```

# Memory

### Memory is used for temporary storage of variables. It is cleared at the end of the function call.

### Memory is used in the following cases:

-   Return values to external calls
-   Set the function arguments for external calls
-   Get values from external calls
-   Revert with an error string
-   Log messages
-   Create other contracts
-   Use the keccak256 function

### Memory is laid out in 32 byte sequences.

## Memory keywords:

-   `mload(p)`: Retrieves 32 bytes from memory from slot p [p .. 0x20]
-   `mstore(p,v)`: Stores 32 bytes from v into memory slot p [p .. 0x20]
-   `mstore8(p,v)`: Like mstore, but only stores 1 byte
-   `msize`: Returns the largest accessed memory index in the current transaction

Using `mstore` 7 into memory slot 0:

```solidity
    assembly {
        // empty memory looks like this:
        //  00   00   00   00  ...  00   00   00   00
        // 0x00 0x01 0x02 0x03 ... 0x17 0x18 0x19 0x20

        mstore(0, 7)
        // is the same as doing:
        // mstore(0, 0x0000...000007) // 32 bytes

        // the memory now looks like this:
        //  00   00   00   00  ...  00   00   07   00
        // 0x00 0x01 0x02 0x03 ... 0x17 0x18 0x19 0x20
    }
```

Using `mstore8` 7 into memory slot 0:

```solidity
    assembly {
        // empty memory looks like this:
        //  00   00   00   00  ...  00   00   00   00
        // 0x00 0x01 0x02 0x03 ... 0x17 0x18 0x19 0x20

        mstore8(0, 7)
        // is the same as doing:
        // mstore8(0, 0x07) // 1 byte

        // the memory now looks like this:
        //  07   00   00   00  ...  00   00   00   00
        // 0x00 0x01 0x02 0x03 ... 0x17 0x18 0x19 0x20
    }
```

## How Solidity uses memory:

-   Solidity allocates slots [0x00-0x20], [0x20-0x40] for "scratch space" (first 32x2 bytes)
-   Solidity reserves slot [0x40-0x60] as the "free memory pointer" (the location of the next free memory slot)
-   Solidity keeps slot [0x60-0x80] empty (next 32 bytes)
-   The action begins at slot [0x80-...]

### To get the next free memory slot in Solidity, so you can use it knowing that it is empty, use the following code:

```solidity
    assembly {
        let freeMemoryPointer := mload(0x40)
    }
```

### Memory Struct

Adding structs to memory is just like adding their values 1 by 1.

```solidity
    struct S {
        uint256 a;
        uint256 b;
    }

    function f() external {
        bytes32 freeMemoryPointer;

        S memory s = S(a: 1, b: 2);

        assembly {
            // free memory pointer is now 0x80 + 32 bytes * 2 = 0xc0
            freeMemoryPointer := mload(0x40)

        // mload(0x80) would return a (1)
        // mload(0xa0) would return b (2)
        }
    }
```

### Memory Fixed Arrays

Fixed arays work just like structs

```solidity
    function f() external {
        uint256[2] memory arr = [1, 2];

        assembly {
            // mload(0x80) would return 0x0000...000001 (32 bytes)
            // mload(0xa0) would return 0x0000...000002 (32 bytes)
        }
    }
```

### Memory Dynamic Arrays

For dynamic arrays, the first memory slot of 32 bytes is used to store the lenght of the array. In Yul, the array value is the location of the array in memory.

```solidity
    function f(uint256[] memory arr) external {
        bytes32 location;
        bytes32 length;

        assembley {
            // the location will be the first free memory pointer: 0x80
            location := arr
            // the length will be the first memory slot of the array: 0x80
            length := arr.length

            mload(add(location, 0x20)) // returns the first element of the array
            mload(add(location, 0x40)) // returns the second element of the array
        }
    }
```

### abi.encode

The operation abi.encode will first push the bytes length of the arguments onto memory and then the arguments. If any argument is smaller than 32 bytes, it will be padded to 32 bytes.

```solidity
    function f() external {
        abi.encode(uint256(1), uint256(2));

        assembly {
            // mload(0x80) would return 0x0000...000040 (the bytes length of the arguments: 64)
            // mload(0xa0) would return 0x0000...000001 (32 bytes)
            // mload(0xc0) would return 0x0000...000002 (32 bytes)
        }
    }
```

### abi.encodePacked

Compared to abi.encode, abi.encodePacked will not add padding to the arguments.

```solidity
    function f() external {
        abi.encodePacked(uint256(1), uint128(2));

        assembly {
            // mload(0x80) would return 0x0000...000030 (the bytes length of the arguments: 48)
            // mload(0xa0) would return 0x0000...000001 (32 bytes)
            // mload(0xc0) would return 0x00...0002 (16 bytes)
        }
    }
```

### Only reading from memory makes msize consider the memory slot as used.

```solidity
    bytes32 slot;
    bytes32 _msize;

    assembly {
        ssembly {
            // read a bite from memory slot 0xff
            pop(mload(0xff))
            // the free memory pointer is still 0x80
            slot := mload(0x40)
            // but msize considers the memory slot 0xff as used
            // so _msize is 0x120
            _msize := msize()
        }
    }
```

### abi.encode and abi.encodePacked

-   When using abi.encode, the compiler will add padding to the data to make it fit into 32 byte slots.

-   When using abi.encodePacked, the compiler will not add padding to the data.

# General Notes

-   In in-lane assambely, you can only assign values to variables on the stack. You cannot assign values to variables in storage or memory.
-   Yul has no overflow protection!
