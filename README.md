# Web3 Security Notes

A comprehensive reference for blockchain security, covering multiple platforms, security findings, and fundamental concepts.

## Repository Structure

```text
web3-security-notes/
├── blockchain/                # Blockchain ecosystems
│   ├── Ethereum/
│   │   ├── EIPs-and-ERCs/    # Token standards, upgrades, signatures, gas
│   │   ├── Solidity/
│   │   │   ├── Language/     # Compiler, memory, ABI encoding, data types
│   │   │   └── Patterns/     # Access control, common contract patterns
│   │   ├── Tooling/          # Foundry, IDE audit tools
│   │   ├── L2/               # Optimism, rollups
│   │   └── Protocols/        # Uniswap, EigenLayer
│   ├── Solana/               # Architecture, Anchor, common bugs
│   └── cosmwasm/             # CosmWasm architecture
├── programming-languages/     # Cross-chain languages
│   └── rust/                 # Memory, macros, error handling, REVM
├── fundamentals/              # Core concepts
│   ├── cryptography/         # BLS, cryptographic primitives
│   ├── networking/           # P2P, Bitcoin networking
│   └── node-operations/      # Sequencers, L2 operations
├── Findings/                  # Real-world vulnerabilities
│   ├── cross-chain/          # Bridge & interop exploits
│   ├── infrastructure/       # Consensus, network, crypto issues
│   └── defi/                 # DeFi protocol vulnerabilities
├── resources/                 # External learning materials
└── tools/                     # Security tools & scripts
```

## Quick Navigation

### Blockchain

- [Ethereum](blockchain/Ethereum/) - EIPs/ERCs, Solidity, Foundry, L2s, protocols
  - [Solidity Language](blockchain/Ethereum/Solidity/Language/) - Compiler, memory, ABI encoding, data types
  - [Solidity Patterns](blockchain/Ethereum/Solidity/Patterns/) - OZ access control, common patterns
  - [EIPs & ERCs](blockchain/Ethereum/EIPs-and-ERCs/) - Token standards, upgrades, signatures
  - [Tooling](blockchain/Ethereum/Tooling/) - Foundry cheat codes, IDE audit tools
  - [Protocols](blockchain/Ethereum/Protocols/) - Uniswap V2, EigenLayer
- [Solana](blockchain/Solana/) - Architecture, Anchor framework, rent, common bugs
- [CosmWasm](blockchain/cosmwasm/) - CosmWasm architecture

### Programming Languages

- [Rust](programming-languages/rust/) - Memory safety, macros, error handling, REVM

### Fundamentals

- [Cryptography](fundamentals/cryptography/) - BLS signatures, cryptographic primitives
- [Networking](fundamentals/networking/) - P2P protocols, Bitcoin networking
- [Node Operations](fundamentals/node-operations/) - Sequencers, L2 infrastructure

### Security Findings

- [Cross-Chain](Findings/cross-chain/) - Bridge exploits, relayer vulnerabilities
- [Infrastructure](Findings/infrastructure/) - Consensus bugs, network attacks
- [DeFi](Findings/defi/) - Protocol-specific vulnerabilities

### Resources

- [External Resources](resources/external-resources.md) - Curated learning materials
