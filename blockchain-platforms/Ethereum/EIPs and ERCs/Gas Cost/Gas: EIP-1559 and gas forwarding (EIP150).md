# Ethereum Gas: Pre-EIP-1559, Post-EIP-1559, and Gas Forwarding (EIP-150)

---

## 1. Pre-EIP-1559 (Legacy Gas Model)

Before the London hard fork (August 2021), Ethereum used a **simple auction-based fee model**:

* A transaction specified:

  * **`gasLimit`** → maximum gas units the tx could consume.
  * **`gasPrice`** → price per gas unit (in wei).

* The total cost paid by the sender was:

  ```text
  totalFee = gasUsed * gasPrice
  ```

* Miners received **all of this fee** as a reward.

* Issues:

  * Fees were volatile because users bid blindly against each other.
  * Miners had incentives to manipulate fees.
  * No ETH was burned — all fees went to miners.

---

## 2. Post-EIP-1559 (London Hard Fork)

EIP-1559 introduced a **dual-component fee model** to improve predictability and align incentives.

### 2.1 New Fields in a Transaction

Users now specify three parameters:

1. **`gasLimit`** → maximum gas units allowed.
2. **`maxFeePerGas`** → ceiling (cap) on price per gas unit the user is willing to pay.
3. **`maxPriorityFeePerGas`** → maximum tip per gas unit to give to the block proposer.

### 2.2 Base Fee Mechanism

* Each block has a **`baseFee`**, which adjusts automatically based on demand:

  * If blocks are >50% full, base fee rises.
  * If blocks are <50% full, base fee falls.

* The **base fee is burned** 🔥 (permanently removed from supply).

### 2.3 Effective Gas Price

At execution, the actual gas price is:

```text
effectiveGasPrice = min(maxFeePerGas, baseFee + maxPriorityFeePerGas)
```

The total cost to the user is:

```text
totalCost = gasUsed * effectiveGasPrice
```

Split into:

* `gasUsed * baseFee` → burned
* `gasUsed * priorityFee` → paid to validator

### 2.4 Effects

* Validators now only earn **priority fees** + issuance + MEV.
* ETH becomes **deflationary** during high usage (since base fees are burned).
* Users enjoy more predictable fees, since they only need to set a tip and a safety ceiling.

---

## 3. Gas Forwarding (EIP-150 Rule)

When contracts call other contracts, **not all remaining gas is forwarded**.
This is controlled by **EIP-150 (the “63/64 rule”)**, introduced in 2016.

* If contract A calls contract B via `.call{}` **without specifying `gas`**,

  * The EVM forwards at most **63/64 of the remaining gas**.
  * The caller always keeps at least 1/64 of the gas for cleanup and safety.

Mathematically:

```text
gasForwarded = gasleft() - gasleft()/64
```

* This prevents griefing attacks where a callee deliberately consumes *all* gas, leaving the caller unable to finish its logic.

### Important Distinction

* **Gas units forwarded** (EIP-150) = how much computation the callee can do.
* **Gas price (per unit)** (EIP-1559) = how much each gas unit costs in ETH.

These are **independent dimensions**:

* Units = `gasLimit` / `63/64 rule`
* Price = `baseFee + tip` under EIP-1559

---

## 4. Putting It All Together

* **Pre-EIP-1559:**

  * Users set `gasLimit` + `gasPrice`.
  * Miners earned all fees.

* **Post-EIP-1559:**

  * Users set `gasLimit`, `maxFeePerGas`, `maxPriorityFeePerGas`.
  * Effective price = `min(maxFee, baseFee + tip)`.
  * Base fee burned, tip → validator.

* **EIP-150 Gas Forwarding Rule:**

  * On contract calls, at most 63/64 of remaining gas is forwarded.
  * Protects callers from being left with zero gas.

---

👉 So in short:

* **GasLimit** protects users from infinite loops.
* **EIP-1559 fees** make pricing predictable and deflationary.
* **EIP-150 forwarding** ensures safe gas handling between contracts.
