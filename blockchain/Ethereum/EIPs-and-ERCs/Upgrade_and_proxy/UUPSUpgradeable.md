# UUPS Upgradeable Proxy

## Overview

The UUPS pattern is an upgradeable contract pattern that moves the upgrade functionality to the implementation contract rather than keeping it in the proxy. This approach addresses some limitations of the Transparent Proxy pattern while maintaining upgradeability.

## Key Components

1. Proxy Contract (minimal)
2. Implementation Contract (includes upgrade logic)
3. ERC1967 Storage Slots

## Core Concept

In UUPS, the upgrade logic resides in the implementation contract. This allows for a simpler proxy contract and potentially lower gas costs for regular function calls.

## Technical Details

### 1. Proxy Contract Structure

```solidity
contract ERC1967Proxy is ERC1967Upgrade {
    constructor(address _logic, bytes memory _data) payable {
        _upgradeToAndCall(_logic, _data, false);
    }

    fallback() external payable virtual {
        _fallback();
    }

    receive() external payable virtual {
        _fallback();
    }

    function _fallback() internal virtual {
        _delegate(_implementation());
    }
}
```

### 2. Implementation Contract Structure

```solidity
abstract contract UUPSUpgradeable is IERC1822Proxiable {
    address private immutable __self = address(this);

    modifier onlyProxy() {
        require(address(this) != __self, "Function must be called through delegatecall");
        _;
    }

    modifier notDelegated() {
        require(address(this) == __self, "UUPSUpgradeable: must not be called through delegatecall");
        _;
    }

    function upgradeTo(address newImplementation) external virtual onlyProxy {
        _authorizeUpgrade(newImplementation);
        _upgradeToAndCall(newImplementation, new bytes(0), false);
    }

    function upgradeToAndCall(address newImplementation, bytes memory data) external payable virtual onlyProxy {
        _authorizeUpgrade(newImplementation);
        _upgradeToAndCall(newImplementation, data, true);
    }

    function _authorizeUpgrade(address newImplementation) internal virtual;
}

```

### 3. Key Functions

- `upgradeTo(address)`: Upgrades the implementation contract.
- `upgradeToAndCall(address, bytes)`: Upgrades and calls an initialization function.
- `_authorizeUpgrade(address)`: Internal function to authorize upgrades (must be overridden, implemented by Child contract).

### 4. ERC1967 Storage Slots

UUPS uses ERC1967 storage slots for storing the implementation address:

```solidity
bytes32 internal constant IMPLEMENTATION_SLOT = 0x360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc;
```

### 5. Upgrade Mechanism

```solidity
function _upgradeToAndCall(address newImplementation, bytes memory data, bool forceCall) internal {
    _setImplementation(newImplementation);
    if (data.length > 0 || forceCall) {
        Address.functionDelegateCall(newImplementation, data);
    }
}
```

## Technical Implications

1. **Gas Efficiency**:
    - No conditional logic in the proxy, potentially lower gas costs for regular calls.
2. **Upgrade Safety**:
    - Upgrade logic is in the implementation, allowing for custom authorization mechanisms.
3. **Simplicity**:
    - Proxy contract is simpler and more lightweight.
4. **Flexibility**:
    - Allows for different upgrade patterns in different implementations.
5. **ERC1967 Compatibility**:
    - Uses ERC1967 storage slots for upgradability.

## Implementation Considerations

1. **Upgrade Authorization**:
    - Implement `_authorizeUpgrade` carefully to ensure proper access control.
2. **Initialization**:
    - Use initializer functions instead of constructors in implementation contracts.
3. **Storage Layout**:
    - Maintain consistent storage layout across upgrades.
4. **Immutability**:
    - Be cautious with `immutable` variables, as they are not part of the storage layout.

## Advantages

1. Potentially lower gas costs for regular function calls.
2. More flexible upgrade mechanisms.
3. Simpler proxy contract.

## Disadvantages

1. Upgrade logic must be included in every implementation.
2. Potential for accidentally removing upgrade functionality.

## Usage Example

```solidity
// Implementation contract
contract MyContractV1 is UUPSUpgradeable {
    uint256 public value;

    function initialize(uint256 _value) public initializer {
        value = _value;
    }

    function _authorizeUpgrade(address newImplementation) internal override {
// Add authorization logic here
        require(msg.sender == owner(), "Not authorized");
    }
}

// Deployment
MyContractV1 implementation = new MyContractV1();
ERC1967Proxy proxy = new ERC1967Proxy(
    address(implementation),
    abi.encodeWithSelector(MyContractV1.initialize.selector, 42)
);

// Interaction
MyContractV1(address(proxy)).value();

// Upgrade
MyContractV1(address(proxy)).upgradeTo(address(new MyContractV2()));
```

## Security Considerations

1. Ensure robust access control in `_authorizeUpgrade`.
2. Be cautious not to remove or break the upgrade functionality in new implementations.
3. Thoroughly test upgrades, including the upgrade process itself.
4. Consider using a timelock for critical upgrades.
5. Be aware of potential vulnerabilities introduced by upgrades.