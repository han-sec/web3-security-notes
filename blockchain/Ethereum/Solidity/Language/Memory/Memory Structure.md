### Reserved Memory Ranges

```jsx
0x00 - 0x20: Scratch space for hashing methods
0x20 - 0x40: Currently allocated memory size
0x40 - 0x60: Free memory pointer
0x60 - 0x80: Zero slot (scratch space for dynamic memory arrays)
0x80 onwards: User-allocated memory starts here
```

## Free Memory Pointer (0x40)

### Purpose

- Points to the next available memory slot
- Always points to a 32-byte word boundary
- Initially set to 0x80 (first non-reserved slot)

### Rules

```solidity
// Reading the free memory pointer
let ptr := mload(0x40)

// Updating the free memory pointer
mstore(0x40, add(ptr, size))
```

## Memory Allocation Pattern

### Basic Parameter Layout

Starting from 0x80:

```jsx
0x80 - 0x9F: First parameter (32 bytes)
0xA0 - 0xBF: Second parameter (32 bytes)
0xC0 - 0xDF: Third parameter (32 bytes)
...and so on
```

### Example Function

```solidity
function example(
    address account,// 20 bytes padded to 32
    uint256 amount,// 32 bytes
    bytes memory data// dynamic size
) external {
		// Memory layout at function start:
		// 0x80: account
		// 0xA0: amount
		// 0xC0: data offset
		// 0xE0 onwards: [32bytes data length] - [actual data bytes]
}
```

## Dynamic Memory Management

### Allocating Dynamic Arrays

```solidity
// For a new bytes array
// 1. Store length at current free memory pointer
// 2. Store data starting at next 32-byte boundary
// 3. Update free memory pointer to after databytes memory newArray = new bytes(length);

Layout:
[ptr]      : length
[ptr+0x20] : data starts here
[ptr+0x20+padded_length] : next free slot
```

### String Allocation

```solidity

// Strings are encoded the same as bytesstring memory str = "Hello";

Layout:
[ptr]      : length (5)
[ptr+0x20] : data ("Hello" padded to 32 bytes)
```

## Complex Example

```solidity
function complexExample(
    uint256 staticVal,
    bytes memory data1,
    bytes memory data2
) external {
		// Initial memory layout:
		// 0x80: staticVal
		// 0xA0: data1 offset
		// 0xC0: data2 offset// 0xE0: data1 length
		// 0x100: data1 content
		// [next 32-byte boundary]: data2 length
		// [next slot]: data2 content// Creating new array
    uint256[] memory newArray = new uint256[](3);
		// Adds:
		// [next free slot]: array length (3)
		// [+0x20]: array data slots (3 * 32 bytes)
}

```

## Memory Expansion and Gas Costs

### Expansion Rules

1. Memory can only expand in 32-byte chunks
2. Gas cost increases quadratically with size
3. Never contracts (only expands)

### Gas Calculation

```jsx
// Gas cost formula for memory expansion:
// 3 gas per byte for first 724 bytes
// Additional bytes cost increases quadratically
```

## Memory Regions During Function Execution

### 1. Input Parameters Region

```
0x80 onwards: Function parameters stored sequentially
```

### 2. Temporary Variables Region

```
[After parameters]: Space for temporary variables
```

### 3. Dynamic Allocation Region

```
[After temp vars]: New memory allocations
```

## Working with Memory - Examples

### 1. Basic Memory Operations

```solidity
function memoryExample() external {
		// Allocate new bytes
    bytes memory data = new bytes(100);
		// Memory layout:
		// [free ptr]: length (100)
		// [free ptr + 0x20]: data (100 bytes padded to nearest 32 bytes)
		// Update free memory pointer to next 32-byte boundary
}
```

### 2. Nested Structures

```solidity
struct Example {
    uint256 value;
    bytes data;
}

function structExample() external {
    Example memory e = Example({
        value: 123,
        data: new bytes(64)
    });
		// Memory layout:
		// [ptr]: value (32 bytes)
		// [ptr+0x20]: offset to data
		// [ptr+0x40]: data length
		// [ptr+0x60]: data content
}

```

## Best Practices

### 1. Memory Safety

```solidity
// Always validate memory ranges
require(ptr + size <= totalMemorySize);

// Check array bounds
require(array.length <= maxLength);
```

### 2. Gas Optimization

- Reuse memory when possible
- Pack similar operations together
- Be aware of memory expansion costs

### 3. Common Patterns

```solidity
// Temporary memory usage
function safePattern() external {
    uint256 oldFreePtr = mload(0x40);
		// ... use memory ...
    require(mload(0x40) > oldFreePtr);
}
```

## Debugging Memory Layout

### Common Issues

1. Memory corruption from incorrect pointer arithmetic
2. Unexpected memory expansion costs
3. Incorrect dynamic array handling

### Debugging Tools

```solidity
// Debug memory layout
function debugMemory(uint256 start, uint256 slots) internal pure {
    for(uint256 i = 0; i < slots; i++) {
        bytes32 value = mload(start + i * 32);
		// Log or process value
    }
}

```

## Assembly Level Memory Access

### Direct Memory Operations

```solidity
assembly {
// Load value from memory
    let value := mload(ptr)

// Store value to memory
    mstore(ptr, value)

// Store single byte
    mstore8(ptr, byte_value)
}
```

### Safe Memory Management

```solidity
function safeMemory() external {
    assembly {
				// Get current free memory pointer
        let ptr := mload(0x40)

				// Ensure 32-byte alignment
        ptr := and(add(ptr, 31), not(31))

				// Update free memory pointer
        mstore(0x40, add(ptr, size))
    }
}
```