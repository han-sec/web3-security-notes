# Cross-chain findings

### **Across V3 – Cross Chain Action Exploit**

A bug in the early return logic allowed a relayer to mark a user's cross-chain action as completed without actually transferring funds. The relayer could then claim the full refund from LP funds, effectively stealing the user's assets. [Read More](https://paragraph.com/@zachobront/across-v3-cross-chain-action-vulnerability-disclosure)

### Scroll – EnforcedTxGateway Message Spoofing

A missing check allowed the gateway to call itself via a relayed L2 message, letting attackers craft calldata that executed privileged actions (e.g., minting tokens) while msg.sender appeared trusted.[Read More](https://forum.scroll.io/t/report-scroll-mainnet-emergency-upgrade-on-2025-04-25/666)
