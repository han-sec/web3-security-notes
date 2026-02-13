# Ethereum Gas Costs (Common Operations)

This list summarizes the most important gas costs defined in the Ethereum Yellow Paper and subsequent EIPs.  
Values may differ slightly between Ethereum upgrades (e.g., Istanbul, London, Shanghai).  

---

## 📦 Transaction & Calldata

| Operation                           | Gas Cost        | Notes |
|-------------------------------------|-----------------|-------|
| Base transaction cost               | **21,000**      | Every transaction |
| Calldata byte (zero)                | **4**           | Per zero byte of calldata |
| Calldata byte (non-zero)            | **16**          | Per non-zero byte of calldata (EIP-2028 reduced from 68 → 16) |

---

## 🗄️ Storage

| Operation                           | Gas Cost        | Notes |
|-------------------------------------|-----------------|-------|
| Storage write (0 → non-zero)        | **20,000**      | First time slot use |
| Storage write (non-zero → non-zero) | **5,000**       | Update existing slot |
| Storage write (non-zero → 0)        | **5,000** (refund **+15,000**) | Net = 5,000 - refund |
| Storage read (`SLOAD`)              | **2,100**       | Load from storage |
| Cold storage access (EIP-2929)      | **2,100 + 2,600** | First access in tx |

---

## 🧠 Memory

| Operation                           | Gas Cost        | Notes |
|-------------------------------------|-----------------|-------|
| Memory expansion                    | **3 gas per word (32 bytes)** + quadratic cost | Cost grows with memory size |
| MLOAD (load from memory)            | **3**           | Per 32-byte word |
| MSTORE (store to memory)            | **3**           | Per 32-byte word |

---

## 🔗 Calls

| Operation                           | Gas Cost        | Notes |
|-------------------------------------|-----------------|-------|
| `CALL` (message call)               | **700** + value transfer costs | Plus 9,000 if sending ETH |
| `DELEGATECALL` / `CALLCODE`         | **700**         | Same as call, but context preserved |
| `STATICCALL`                        | **700**         | No state changes allowed |
| Call with value transfer            | **+9,000**      | Added if ETH is transferred |
| New account creation (via call)     | **+25,000**     | Extra cost if address didn’t exist |

---

## 📜 Contract Creation & Code

| Operation                           | Gas Cost        | Notes |
|-------------------------------------|-----------------|-------|
| Contract creation base cost         | **32,000**      | Includes deploying code |
| Code deposit                        | **200 gas per byte** | Cost per byte of contract code |

---

## 🔍 Execution / Arithmetic

| Operation                           | Gas Cost        | Notes |
|-------------------------------------|-----------------|-------|
| `ADD`, `SUB`                        | **3**           | Simple arithmetic |
| `MUL`                               | **5**           | Multiplication |
| `DIV`, `MOD`                        | **5**           | Division |
| `EXP`                               | **10 + 50 per byte of exponent** | Exponentiation |
| `SHA3` (Keccak-256 hash)            | **30 + 6 per word (32 bytes)** | Hash cost |

---

## ⛽ Other Important Costs

| Operation                           | Gas Cost        | Notes |
|-------------------------------------|-----------------|-------|
| Log operation (`LOG0` - `LOG4`)     | **375 + 375 per topic + 8 per byte** | Event emission |
| Self-destruct (`SELFDESTRUCT`)      | **5,000** (refund **+24,000**) | Removes contract, refunds gas |
| Balance check (`BALANCE`)           | **100**         | Per call |
| External code size (`EXTCODESIZE`)  | **100** (warm) / **2,600** (cold) | Check contract code size |
| External code copy (`EXTCODECOPY`)  | **100 + 3 per word** | Copy contract code |

---

📖 **References:**

- [Ethereum Yellow Paper](https://ethereum.github.io/yellowpaper/paper.pdf)  
- [EIP-2028: Calldata Cost Reduction](https://eips.ethereum.org/EIPS/eip-2028)  
- [EIP-2929: Gas Cost Increases for State Access](https://eips.ethereum.org/EIPS/eip-2929)  
- [EIP-3529: Gas Refund Reductions](https://eips.ethereum.org/EIPS/eip-3529)  

---

✅ This table focuses on the **most common gas costs** auditors and developers need when reasoning about performance and DoS vectors.
