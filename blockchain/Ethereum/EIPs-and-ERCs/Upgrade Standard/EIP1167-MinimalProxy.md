# EIP-1167: Minimal Proxy Contract

## Overview

EIP-1167 standardizes the implementation of minimal proxy contracts, also known as `clones`, which delegate all calls to a pre-deployed implementation contract. This approach offers a gas-efficient mechanism to deploy multiple instances of a contract with identical functionality.

## Motivation

Deploying multiple identical smart contracts can be expensive due to gas costs. The minimal proxy pattern addresses this by creating lightweight proxy contracts that delegate calls to a master implementation contract. This significantly reduces deployment costs and minimizes gas overhead during execution.

## Key Features

### Standardized Bytecode

The minimal proxy contract has a fixed bytecode structure. The core functionality is encoded as:

```solidity
0x363d3d373d3d3d363d73 <address> 5af43d82803e903d91602b57fd5bf3
```

- Bytes 10-29: Contain the `20-byte address` of the implementation contract(added call into proxy will be delegated to `implementation contract`).
- Functionality: Delegates all calls to the implementation contract, forwarding all gas and returning any data or errors from the delegate call.

```solidity
Proxy Bytecode Structure:
[EIP-1167 Proxy Code (minimal proxy logic)][Immutable Args]
```

### Simplicity and Gas Efficiency

Low Deployment Cost: By deploying minimal bytecode, the gas cost is significantly reduced compared to deploying full contract code.

### Locked-Down Behavior

Minimal proxies are immutable and lack upgradability. This ensures predictable behavior tied directly to the implementation contract.

### Specification

The standardized bytecode performs the following operations:

- Delegation: Forwards all function calls, along with their calldata, to the implementation contract.
- Return Handling: Returns the output from the implementation contract to the caller.
- Error Propagation: Reverts with the original error message if the implementation contract reverts.

### Example Proxy Bytecode

Disassembly of the minimal proxy bytecode:

1. **Opcode**

```solidity
36 calldatasize
3d returndatasize
37 calldatacopy
73 push20 (address) // get implementation address
5a gas
f4 delegatecall // delegate call to implementation contract
3d returndatasize
3e returndatacopy
f3 return
fd revert
```

### Vanity Address Optimization

Proxy deployment can be further optimized by using an implementation contract with a vanity address. This reduces the size of the proxy bytecode by shortening the push instruction and decreasing its gas cost.

### Advantages

1. Cost-Effective: Significant gas savings during both deployment and execution.
2. Reusability: Centralized implementation contracts reduce duplication and simplify updates to functionality.
3. Predictability: Immutable proxies ensure that functionality remains consistent and tamper-proof.

### Use Cases

1. Token Factories: Deploying multiple tokens with identical logic.
2. DeFi Protocols: Cloning contracts for lending pools or staking mechanisms.
3. NFT Projects: Generating identical smart contracts for collections or marketplaces.