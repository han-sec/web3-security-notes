# Cross-Chain Gas & Fee Mechanics

Understanding how gas fees work across different chains is critical for secure bridge integration. This document breaks down the two primary models used in cross-chain communication: the **Push Model** (e.g., Wormhole) and the **Pull Model** (e.g., Optimism/Arbitrum).

## 1. The Core Concept: "Who Pays the Postman?"

Every cross-chain transaction involves two costs:

1. **Source Chain Gas:** The cost to send the message *out*.
2. **Destination Chain Gas:** The cost to execute the message *in*.

The major difference between bridges is **when** and **who** pays for the Destination Chain Gas.

---

## 2. L1 $\to$ L2 (Deposits)

**Model:** Push (Pay Upfront)

When sending funds or data from Ethereum (L1) to a Layer 2 (L2), the user pays for everything upfront. The L1 Bridge calculates the estimated cost of the L2 transaction and charges the user immediately.

### Flow Diagram

```mermaid
sequenceDiagram
    actor User
    participant L1_Dispenser as L1 Dispenser
    participant L1_Bridge as L1 Bridge
    participant L2_Bridge as L2 Bridge (Sequencer)
    participant L2_Target as L2 Target

    Note over User, L1_Dispenser: Step 1: Upfront Payment
    User->>L1_Dispenser: Call deposit() + Attach ETH (Value + L2 Gas)
    
    L1_Dispenser->>L1_Bridge: Forward Message + ETH
    Note right of L1_Bridge: Bridge locks ETH &<br/>records "Retryable Ticket"

    Note over L1_Bridge, L2_Bridge: ... Time passes (Minutes) ...

    Note over L2_Bridge, L2_Target: Step 2: Auto-Execution
    L2_Bridge->>L2_Target: Execute Message (Uses Upfront ETH)
    
    alt Gas Price Spikes?
        L2_Bridge--xL2_Target: Revert (Out of Gas)
        Note right of L2_Target: Message is saved in "Retry Queue".<br/>User can replay manually.
    else Normal Execution
        L2_Target-->>L2_Target: Success
    end

```

### Key Takeaways

* **User Pays:** L1 Transaction Fee + L2 Execution Fee (bundled in `msg.value`).
* **Safety Net:** If gas prices spike on L2 and the transaction fails, the message is **not lost**. It sits in a "Retry Queue" on L2, where the user can manually provide more gas to push it through.

---

## 3. L2  L1 (Withdrawals / Sync)

This direction varies significantly depending on the bridge architecture.

### Scenario A: Native Rollups (Arbitrum, Optimism)

**Model:** Pull (Pay Later / Collect-on-Delivery)

These bridges do **not** automatically execute the transaction on L1. They simply "post" the message to L1. The user (or a bot) must claim it later.

* **L2 Fee:** Cheap. Just recording the message.
* **L1 Fee:** Expensive. Paid by the user 7 days later.

```mermaid
sequenceDiagram
    actor User
    participant L2_Contract as L2 Contract
    participant L2_Sys as ArbSys / OVM
    participant L1_Outbox as L1 Outbox
    
    Note over User, L2_Contract: Step 1: Queue Message (Cheap)
    User->>L2_Contract: syncWithheldAmount{value: 0.1 ETH}()
    
    Note right of L2_Contract: Native bridges don't charge ETH<br/>to send a message.
    L2_Contract->>L2_Sys: sendTxToL1()
    L2_Contract-->>User: Refund 0.1 ETH (leftovers)
    
    Note over L2_Sys, L1_Outbox: ... Wait 7 Days (Challenge Period) ...
    
    Note over User, L1_Outbox: Step 2: Claim Message (Expensive)
    User->>L1_Outbox: executeTransaction()
    Note right of User: User pays L1 Gas directly here.
    L1_Outbox->>L1_Outbox: Success

```

### Scenario B: Generic Bridges (Wormhole, LayerZero)

**Model:** Push (Pay Upfront / FedEx)

These bridges use third-party "Relayers" to execute the transaction on the destination chain for you. You must pay the Relayer upfront on the source chain.

```mermaid
sequenceDiagram
    actor User
    participant L2_Contract as L2 Contract
    participant Wormhole as Wormhole Relayer
    participant L1_Contract as L1 Contract
    
    Note over User, L2_Contract: Step 1: Pay Relayer (Full Cost)
    User->>L2_Contract: syncWithheldAmount{value: 0.1 ETH}()
    
    Note right of L2_Contract: 1. Get Quote (e.g. 0.05 ETH)<br/>2. Pay Relayer
    L2_Contract->>Wormhole: sendPayload{value: 0.05 ETH}()
    L2_Contract-->>User: Refund 0.05 ETH (Change)
    
    Note over Wormhole, L1_Contract: ... Wait ~15 Minutes ...
    
    Note over Wormhole, L1_Contract: Step 2: Auto-Execution
    Wormhole->>L1_Contract: Execute Message
    Note right of Wormhole: Relayer pays L1 Gas<br/>using the 0.05 ETH collected earlier.

```

---

## 4. Summary Table

| Feature | **Native Rollups (Arb/Op)** | **Generic Bridges (Wormhole)** |
| --- | --- | --- |
| **Direction** | L2  L1 | L2  L1 (or L2  L2) |
| **Model** | **Pull** (Manual Claim) | **Push** (Auto Delivery) |
| **Who executes?** | **You** (User/Keeper) | **Relayer** (3rd Party) |
| **When do you pay?** | **Later** (on L1) | **Now** (on L2) |
| **`msg.value` Logic** | `leftovers = msg.value` (Full Refund) | `leftovers = msg.value - cost` |
| **Gas Risk** | You control the gas limit when you claim. | Requires `Min/Max Gas` checks to protect Relayer. |

---
