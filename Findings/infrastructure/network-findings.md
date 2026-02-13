# Blockchain / Network Findings

A curated list of interesting blockchain/network protocol issues for learning and reference.

## Consensus / Chain Integrity Issues

### **Movement Full Node – Permanent Chain Split**

Missing block-height check allowed two blocks at the same height, causing a permanent fork and requiring a hard fork. [Read More](https://medium.com/@yemresaritoprak/permanent-chain-split-in-movement-full-node-anatomy-of-a-6-710-critical-vulnerability-that-fa75fe66a0c7)

### **Story Network – Double Burn Bug**  

Delegating and withdrawing multiple times in the same block could over-burn tokens, causing validator panic. [Read More](https://www.story.foundation/blog/story-network-postmortem)

## Denial of Service

### **Sui Network – Temporary Total Network Shutdown**

A vulnerability in Narwhal’s `GetCertificates` handler allowed a single request with a large number of digests to crash validator nodes, causing a temporary total network shutdown. [Read More](https://immunefi.com/blog/bug-fix-reviews/sui-network-shutdown/)

## Eigenlayer Audit

1. A fully slashed operator `(slashing factor = 0)` in beaconChainETHStrategy cannot recover ETH deposited after the slashing event. Due to a `slashingFactor != 0` check in `increaseDelegatedShares`, post-slashing deposits cannot be accounted for, creating a permanent lock of otherwise unslashed funds. [Read More](https://cantina.xyz/code/e7af4986-183d-4764-8bd2-1d6b47f87d99/findings/405)

2. Newly deposited native ETH in an EigenPod is unfairly slashed because the protocol applies past AVS penalties to the total contract balance during a checkpoint. This happens because the system bundles "fresh" deposits with "old" stake into a single calculation, applying a global scaling factor reduction to both simultaneously. Consequently, the logic fails to separate clean capital from previously slashed funds, leading to an immediate and unintended loss for the depositor. [Read More](https://cantina.xyz/code/e7af4986-183d-4764-8bd2-1d6b47f87d99/findings/245)
