# Solidity Storage Layout: Mappings and Arrays

## Basic Storage Slots

- Each storage slot is `32 bytes`
- State variables are allocated sequentially starting from slot 0
- Multiple variables can pack into one slot if they fit

## Mappings

```solidity
mapping(address => uint256) balances;
```

- No fixed storage slot for entire mapping
- Individual values stored at: `keccak256(key || slot)`
  - `key`: Mapping key (left padded to 32 bytes)
  - `slot`: Original mapping slot number (right padded to 32 bytes)

Example:

```solidity
// If balances is at slot 1
balances[0x123...] stored at: keccak256(0x123... || 0x01)
```

## Arrays

```solidity
uint256[] numbers;
```

- Array length stored at array's slot (n)
- Array elements stored sequentially starting at: keccak256(n)
- Each element gets its own slot: keccak256(n) + index

Example for dynamic array at slot 2:

```solidity
Length: slot 2
Element 0: keccak256(2) + 0
Element 1: keccak256(2) + 1
```

## Nested Mappings

```solidity
mapping(address => mapping(uint256 => bool)) nested;
```

- Values stored at: `keccak256(value2 || keccak256(value1 || slot))`
- Each level adds another hash operation

## Array of Mappings

```solidity
mapping(uint256 => uint256)[] arrayOfMaps;
```

- Length at slot n
- Each mapping stored at: keccak256(key || (keccak256(n) + index))

This layout ensures:

- Deterministic storage locations
- Collision resistance
- Gas efficient access
- Secure data isolation

Note: All padding operations are:

- Left padding for keys (addresses, uints)
- Right padding for slots