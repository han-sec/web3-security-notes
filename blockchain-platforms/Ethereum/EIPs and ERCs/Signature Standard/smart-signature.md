# Smart Signature Verification Pattern

This document outlines a flexible signature verification pattern commonly used in advanced smart contract systems (e.g., Gnosis Safe, Autonolas).

Instead of relying solely on standard ECDSA logic, this pattern overloads the signature's recovery identifier (`v`) to act as a **Mode Switch**. This allows a single function to support standard EOAs, legacy wallets, and Smart Contract Wallets (Multisigs/DAOs) seamlessly.

## 1. The "Mode Switch" Mechanism

Standard Ethereum signatures use `v` (27 or 28) to recover the public key. This pattern intercepts `v` to route verification logic before attempting recovery.

| `v` Value | Mode | Signer Type | Verification Logic |
| :--- | :--- | :--- | :--- |
| **0, 1** | **Legacy Fix** | Old Hardware Wallets | Auto-correct to 27/28, then standard ECDSA. |
| **27, 28** | **Standard** | EOAs (MetaMask) | Standard ECDSA `ecrecover`. |
| **4*** | **Contract** | Smart Contract Wallets | Delegates to `isValidSignature` (EIP-1271). |
| **5*** | **Pre-Approved** | Trusted Systems | Checks an on-chain allowlist. |

*> *Note: The specific values for Contract/Pre-Approved modes (4/5) are implementation-specific choices. Gnosis Safe uses 0 and 1 for these modes.*

---

## 2. Verification Modes

### Mode A: Standard & Legacy (EOA)
**Trigger:** `v ∈ {0, 1, 27, 28}`

Handles standard users signing off-chain messages via private keys.
* **Legacy Patch:** If `v` is 0 or 1 (pre-EIP-155), add 27 to normalize it to the EVM standard.
* **Execution:** Use `ecrecover(hash, v, r, s)` to derive the signer address.

### Mode B: Smart Contract Wallets (EIP-1271)
**Trigger:** `v == 4` (or custom flag)

Supports wallets that cannot sign messages mathematically (e.g., Gnosis Safe, DAOs).
* **Address Encoding:** The **Wallet Address** is encoded into the `r` field of the signature bytes, as there is no cryptographic `r` value.
* **Execution:** The contract casts `r` to an address and calls `IERC1271.isValidSignature(hash, signature)`.

### Mode C: Pre-Approved Hashes
**Trigger:** `v == 5` (or custom flag)

Allows execution without a new signature if the signer has previously submitted an on-chain transaction to "whitelist" a specific hash.
* **Address Encoding:** The **Signer Address** is encoded into the `r` field.
* **Execution:** Verifies `mapping(signer => mapping(hash => bool))` is `true`.

---

## 3. Handling Dynamic Data (EIP-712 Optimization)

When using EIP-712 (Typed Data) with complex dynamic arrays (e.g., a list of 50 addresses), verification can become gas-prohibitive.

**The "Hash-then-Sign" Pattern:**
Instead of defining the EIP-712 struct with full arrays, define it with a single `bytes32` hash slot.

**Schema:**
`MyAction(bytes32 dataHash, uint256 nonce)`

**Encoding:**
The off-chain client hashes the arrays using `abi.encode` (to preserve length safety) before signing:
```solidity
bytes32 dataHash = keccak256(abi.encode(array1, array2));