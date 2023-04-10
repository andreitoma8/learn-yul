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
            for { } lt(i, n) {} {
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
                // if true, do something
            }
        }
    }

    function ifFalse(uint256 n) public {
        assembly {
            if 0 {
                // if false, do something
            }
        }
    }

    function negation(uint256 n) public {
        assembly {
            // if 0 is 0 result in true
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
    uint128 a = 4;
    uint96 b = 6;
    uint16 c = 8;
    uint8 d = 1;

    function set(uint16 _c) public {
        assembly{
            // Get the storage slot of the variable
            let wholeSlot := sload(c.slot)

            // Clear the variable's bits in the slot
            let cleared := and(wholeSlot, 0xffff0000ffffffffffffffffffffffffffffffffffffffffffffffffffffffff)

            // Shift the new value to the right by the offset of the variable
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
            let wholeSlot := sload(a.slot)

            // Shift the slot to the right by the offset of the variable
            let shifted := shr(mul(a.offset, 8), wholeSlot)

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

For arrays of mappings of variables smaller than 32 bytes, the compiler will pack the variables into a single slot.

# General Notes

-   In in-lane assambely, you can only assign values to variables on the stack. You cannot assign values to variables in storage or memory.
-   Yul has no overflow protection!
