# Web3 Security Notes

A comprehensive reference for blockchain security, covering multiple platforms, security findings, and fundamental concepts.

## Repository Structure

```text
web3-security-notes/
├── blockchain-platforms/      # Blockchain ecosystems
│   ├── ethereum/             # EVM, Foundry, EIPs/ERCs, L2s
│   ├── solana/              # Anchor, eBPF, Solana architecture
│   └── cosmwasm/            # CosmWasm, Cosmos SDK, IBC
├── programming-languages/     # Language-specific security
│   ├── solidity/            # Solidity compiler, memory, ABI
│   └── rust/                # Rust security patterns, REVM
├── fundamentals/             # Core concepts
│   ├── cryptography/        # BLS, cryptographic primitives
│   ├── networking/          # P2P, LibP2P, network protocols
│   └── node-operations/     # Sequencers, L2 operations
├── findings/                 # Real-world vulnerabilities
│   ├── cross-chain/         # Bridge & interop exploits
│   ├── infrastructure/      # Consensus, network, crypto issues
│   └── defi/               # DeFi protocol vulnerabilities
├── tools/                    # Security tools & scripts
└── resources/               # External learning materials
```

## Quick Navigation

### Blockchain Platforms

- [Ethereum](blockchain-platforms/ethereum/) - EVM, Foundry, EIPs/ERCs, L2s, protocols
- [Solana](blockchain-platforms/solana/) - Architecture, Anchor framework, rent mechanism
- [CosmWasm](blockchain-platforms/cosmwasm/) - Cosmos SDK, entry points, cross-contract messages

### Programming Languages

- [Solidity](programming-languages/solidity/) - Compiler, memory structure, ABI encoding, data types
- [Rust](programming-languages/rust/) - Memory safety, macros, error handling, REVM

### Fundamentals

- [Cryptography](fundamentals/cryptography/) - BLS signatures, cryptographic primitives
- [Networking](fundamentals/networking/) - P2P protocols, LibP2P, Bitcoin networking
- [Node Operations](fundamentals/node-operations/) - Sequencers, L2 infrastructure

### Security Findings

- [Cross-Chain](Findings/cross-chain/) - Bridge exploits, relayer vulnerabilities
- [Infrastructure](Findings/infrastructure/) - Consensus bugs, network attacks
- [DeFi](Findings/defi/) - Protocol-specific vulnerabilities

### Resources

- [External Resources](resources/external-resources.md) - Curated learning materials

## Getting Started

1. Browse [blockchain-platforms/](blockchain-platforms/) for platform-specific security knowledge
2. Review [findings/](Findings/) for real-world vulnerability case studies
3. Study [fundamentals/](fundamentals/) for core security concepts
4. Check [resources/](resources/) for external learning materials

## Contributing

When adding new content:

- Place platform-specific notes in the appropriate blockchain-platforms folder
- Document security findings in the findings folder by protocol type
- Add fundamental concepts to the fundamentals folder
- Include references and sources where applicable
