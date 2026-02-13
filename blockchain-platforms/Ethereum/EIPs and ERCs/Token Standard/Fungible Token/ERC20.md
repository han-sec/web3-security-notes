# ERC-20 Token Standard

**Overview:**

- The ERC20 token standard is one of the most widely used standards on the Ethereum blockchain, defining a set of functions and events for creating fungible tokens.
- Commonly used for tokenizing assets, currencies, or other assets.

**Key Functions:**

- `transfer`: Moves tokens from the sender’s account to a recipient.
- `approve`: Allows a spender to withdraw a certain amount of tokens from the sender's account.
- `transferFrom`: Enables transfers from one account to another, typically using allowances.
- `balanceOf`: Returns the balance of a specific account.
- `allowance`: Shows the remaining number of tokens that a spender is allowed to spend on behalf of the owner.

**Limitations:**

- **No Safety Mechanisms:** The basic ERC20 token standard doesn't handle situations where token transfers may fail silently. It assumes all tokens conform to the standard, which is not always the case in practice.
- **Lack of Robustness:** Some ERC20 tokens don’t return `true/false` values or any value at all, which can lead to unexpected failures.

---

### **1.1. What is SafeERC20?**

**Overview:**

- `SafeERC20` is a library provided by OpenZeppelin or Solady to interact with ERC20 tokens safely.
- It adds an extra layer of safety by handling non-standard ERC20 implementations that may fail silently or not return a success value (`true/false`).

**How It Works:**

- Wraps ERC20 function calls (e.g., `transfer`, `transferFrom`, `approve`) in low-level calls to catch and handle failed transactions or unexpected responses.
- If an ERC20 token doesn’t return a value, `SafeERC20` will assume a successful transfer, but it will revert if an error occurs.

**Key Features:**

- **Error Handling:** Uses `require` statements to ensure that transactions either succeed or revert with an error.
- **Gas Efficiency:** Optimized to handle calls more efficiently than manually handling errors.

**Use Case:**

- Use `SafeERC20` in regular, non-upgradeable contracts to ensure robustness and safety when interacting with ERC20 tokens.

---

### **1.2. What is SafeERC20Upgradeable?**

**Overview:**

- `SafeERC20Upgradeable` is the upgradeable version of `SafeERC20`. It’s designed for use in contracts that need to be upgradeable using proxy patterns.
- It ensures compatibility with OpenZeppelin's upgradeable contract library.

**How It Works:**

- It follows the same principles as `SafeERC20` but is adapted to work within the upgradeable environment (`@openzeppelin/contracts-upgradeable`).

**Key Differences from SafeERC20:**

- Uses upgradeable dependencies (e.g., `AddressUpgradeable.sol`) instead of their non-upgradeable counterparts.
- Compatible with contracts that don't use constructors but rely on `initializer` functions due to their upgradeable nature.

**Use Case:**

- Use `SafeERC20Upgradeable` in any upgradeable contract that interacts with ERC20 tokens to maintain safety while allowing contract logic upgrades.

---

### **1.3. Comparing ERC20 vs SafeERC20 vs SafeERC20Upgradeable**

| **Feature/Aspect** | **ERC20** | **SafeERC20** | **SafeERC20Upgradeable** |
| --- | --- | --- | --- |
| **Primary Purpose** | Token standard | Safe interaction with ERC20 tokens | Safe interaction with ERC20 tokens in upgradeable contracts |
| **Error Handling** | Limited | Uses `require` statements to handle errors | Uses `require` statements, adapted for upgradeable environments |
| **Upgradeable Support** | ❌ Not upgradeable | ❌ Not upgradeable | ✅ Designed for upgradeable contracts |
| **Dependencies** | None (standard functions) | Standard OpenZeppelin libraries | Upgradeable OpenZeppelin libraries |
| **Gas Efficiency** | Standard ERC20 efficiency | Slightly higher gas cost due to safety | Slightly higher gas cost due to safety & upgradeability |

---

### **1.4. When to Use Each One**

- **ERC20**:
    - Use when building a basic, standard token without any extra safety measures.
    - When creating simple tokens or interacting with tokens that are guaranteed to conform to the ERC20 standard.
- **SafeERC20**:
    - Use in contracts that interact with ERC20 tokens in a **non-upgradeable** context.
    - Essential when dealing with third-party tokens to ensure robustness and handle potential silent failures.
- **SafeERC20Upgradeable**:
    - Use in **upgradeable contracts** (e.g., those using proxy patterns).
    - Suitable when working with ERC20 tokens in a contract that may evolve over time, ensuring the safety of token interactions.

---

### **1.5. Practical Tips and Best Practices**

- **Always Use `SafeERC20`/`SafeERC20Upgradeable` for Interacting with ERC20s**: These libraries protect your contracts from unexpected behaviors of non-standard tokens eg. Some token did not return `bool` value
- **Upgradeability Considerations**:
    - If you plan on making your contract upgradeable, always use `SafeERC20Upgradeable` from the start.
    - Remember that upgradeable contracts require `initializer` functions instead of constructors.
- **Testing**: When testing your contracts, make sure to include tokens that don’t strictly conform to the ERC20 standard to ensure `SafeERC20`/`SafeERC20Upgradeable` handles them correctly.

---