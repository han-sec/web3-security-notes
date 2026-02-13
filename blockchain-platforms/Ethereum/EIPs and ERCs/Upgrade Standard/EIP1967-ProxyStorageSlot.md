# EIP-1967 Proxy Storage Slot

## Overview

EIP1967 standardizes three critical storage slots for proxy contracts:

1. Implementation Slot
2. Admin Slot
3. Beacon Slot

These slots are used to store addresses crucial for the proxy's operation and upgradability.

## Storage Slot Calculations

The slots are not arbitrary values but are calculated to minimize collision risks:

```solidity

bytes32 constant IMPLEMENTATION_SLOT = bytes32(uint256(keccak256('eip1967.proxy.implementation')) - 1);
//0x360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc;

bytes32 constant ADMIN_SLOT = bytes32(uint256(keccak256('eip1967.proxy.admin')) - 1);
// 0xb53127684a568b3173ae13b9f8a6016e243e63b6e7921f578540a3bc1d3a89fc;

bytes32 constant BEACON_SLOT = bytes32(uint256(keccak256('eip1967.proxy.beacon')) - 1);
// 0xa3f0ad74e5423aebfd80d3ef4346578335a9a72aeaee59ff6cb3582b35133d50
```

### 1. Implementation Slot

- **Value**: `0x360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc`
- **Purpose**: Stores the address of the current implementation contract.

### 2. Admin Slot

- **Value**: `0xb53127684a568b3173ae13b9f8a6016e243e63b6e7921f578540a3bc1d3a89fc`
- **Purpose**: Stores the address of the proxy admin (entity with upgrade rights) mainly for Transparent Proxy.

### 3. Beacon Slot

- **Value**: `0xa3f0ad74e5423aebfd80d3ef4346578335a9a72aeaee59ff6cb3582b35133d50`
- **Purpose**: Stores the address of the beacon contract in the beacon proxy pattern.

## Technical Implications

1. **Collision Avoidance**:
    - The use of keccak256 and subtraction by 1 significantly reduces the chance of colliding with other storage slots.
    - This allows the proxy contract to have its storage without interfering with the implementation contract's storage.

## Important Notes

- EIP1967 only standardizes these storage slots. It does not define how upgrades should be performed or how the admin should be managed.
- The actual implementation of upgrade mechanisms, access control, and proxy patterns are built on top of this storage standardization.
- When implementing EIP1967-compliant proxies, it's crucial to use these exact slot values to ensure compatibility and security.