## 🔹 Withdrawal Flow (Optimism)

### 1. Initiate on L2

* **User → L2StandardBridge**

  * Call: `withdraw(address l1Token, uint256 amount, uint32 l2Gas, bytes calldata data)`
  * Effect: burns/escrows tokens on L2.
  * Internally calls → **L2CrossDomainMessenger** → `sendMessage()`
  * Records withdrawal in **L2ToL1MessagePasser**.

---

### 2. Prove on L1

* **User → L1CrossDomainMessenger**

  * Call: `proveWithdrawalTransaction(withdrawalTx, l2OutputIndex, proof)`
  * Inputs:

    * Encoded withdrawal tx (from step 1)
    * Index of the L2OutputRoot (posted by the **L2OutputOracle**)
    * Merkle proof of inclusion
  * Effect: marks withdrawal as proven. No funds released yet.

---

### 3. Finalize on L1

* **User → L1CrossDomainMessenger**

  * Call: `finalizeWithdrawalTransaction(withdrawalTx)`
  * Checks:

    * Withdrawal was proven earlier
    * 7-day challenge period for the output root has passed
  * If valid → calls the original target (e.g. **L1StandardBridge**)
  * **L1StandardBridge** transfers the withdrawn tokens/ETH to the user.

---

## 🔹 Short Write-Up

On Optimism, withdrawals are a **three-step process** to account for fraud proofs and the challenge period.

1. **Initiate Withdrawal (L2)** → The user calls the `withdraw` function on the L2 bridge. This locks/burns their funds on L2 and records a message to be relayed to L1.

2. **Prove Withdrawal (L1)** → After the corresponding L2 output root is published to L1, the user must “prove” the withdrawal by submitting a Merkle proof to the L1CrossDomainMessenger. This establishes that the withdrawal was indeed included in L2 state.

3. **Finalize Withdrawal (L1)** → Once the 7-day challenge window has passed (so no fraud proof can revert the root), the user finalizes the withdrawal. The L1 bridge then releases the funds.

Thus, the timeline is: **Withdraw → Prove → (wait 7 days) → Finalize.**

---

## 🔹 Comparison with Arbitrum

* **Conceptually the same**:

  * Withdrawals are also delayed until a challenge window passes.
  * Users must provide proof that their withdrawal was included in the L2 chain.

* **Differences**:

  * Arbitrum uses a single **outbox entry** system. Instead of a `prove` + `finalize` split, the proving step is implicitly tied to calling the outbox with an inclusion proof.
  * The challenge window is enforced by delaying when an outbox entry can be executed.
  * Optimism requires two distinct calls (`proveWithdrawalTransaction` and later `finalizeWithdrawalTransaction`).
  * In Arbitrum, once the challenge period passes, you can execute the outbox proof in one shot to finalize the withdrawal.

---

✅ So the **high-level principle is the same**:
Funds are only released on L1 once the L2 assertion they rely on has passed the fraud-proof window.
The **difference is in mechanics**: Optimism has an explicit **Prove → Finalize** split, while Arbitrum bundles proof + finalize into **executeOutbox** after the delay.
