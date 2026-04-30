# ERC-5202: Blueprint Contract Format Implementation

## Overview

ERC-5202 defines a standard format for `blueprint` contracts - contracts that store initcode (initialization code) on-chain. This implementation helps reduce deployer contract size by storing initcode as blueprint contracts that can be retrieved using `EXTCODECOPY`.

## Key Features

### Preamble Format

- Magic bytes: `0xFE71`
- Version bits: 6 bits starting from 0 (0b000000)
- Length encoding: 2 bits (0-2 for length bytes, 3 is reserved)
- Structure: `0xFE71<version bits><length encoding bits><length bytes><data><initcode>`

### Requirements

- Must start with the specified preamble
- Must contain at least one byte of initcode
- Can include optional data between version byte(s) and initcode

## Usage Example

Simple blueprint contract with just a STOP instruction:

```solidity
0xFE710000
```

Blueprint with data section (7 bytes of 0xFF) and STOP instruction:

```solidity
0xFE710107FFFFFFFFFFFFFF00
```

## Implementation Notes

1. The preamble starts with `INVALID (0xFE)` to prevent direct execution
2. Magic byte `0x71` is derived from `keccak256(b"blueprint")[-1]`
3. Version bits allow for future upgrades
4. Length encoding supports variable-length data sections

## Security Considerations

- Blueprint contracts should not be called directly
- Preamble prevents accidental execution of initcode
- Contracts must be verified to follow the ERC-5202 format

## Testing

Basic validation can be performed by checking:

1. Correct magic bytes (0xFE71)
2. Valid version and length encoding
3. Presence of initcode
4. Proper data section length (if present)

## Blueprint vs Normal Deployment

### Normal Contract Deployment

1. **Direct Deployment**
   - Contract bytecode is sent directly in the transaction data
   - Each deployment includes full initialization code
   - Higher gas costs for multiple deployments of the same contract
   - Deployment code is not reusable on-chain

### Blueprint Contract Deployment

1. **Two-Step Process**
   - First: Deploy the blueprint contract (stores initialization code on-chain)
   - Second: Use EXTCODECOPY to copy the initialization code and CREATE/CREATE2 to deploy
   
2. **Key Benefits**
   - Reduced gas costs for multiple deployments (initialization code stored once)
   - Reusable deployment code across multiple contracts
   - More predictable deployment addresses with CREATE2
   
3. **Use Cases**
  - Factory patterns deploying many instances of the same contract
  - Systems requiring deterministic deployment addresses
  - Gas optimization for multiple contract deployments