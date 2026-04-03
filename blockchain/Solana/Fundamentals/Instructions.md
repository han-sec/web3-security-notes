# Reference: Solana Instruction Introspection & Memory Layout

In Solana, programs can "look" at other instructions within the same transaction using the **Instructions Sysvar**. This is a fundamental pattern for atomic operations (like Flashloans), ensuring that if a "Borrow" happens at Index N, a "Payback" must exist at Index N+M.

## 1. Instructions Sysvar Layout (The Container)

The `Sysvar1nstructions1111111111111111111111111` account is a read-only buffer populated by the runtime before a transaction executes.

| Offset | Type | Field | Description |
| :--- | :--- | :--- | :--- |
| `0` | `u16` | `num_instructions` | Total number of instructions in the transaction. |
| `2` | `[u16; N]` | `offset_table` | Array of pointers to the start of each instruction's data. |
| `Varies` | `Bytes` | `body` | Serialized bytes for every instruction in the transaction. |
| `len - 2` | `u16` | `current_index` | The index of the instruction currently executing (Footer). |

---

## 2. Serialized Instruction Layout (The "Instruction" Struct)

When you call `load_instruction_at_checked`, the runtime deserializes a slice of the Sysvar body into this format:

| Field | Size | Type | Description |
| :--- | :--- | :--- | :--- |
| **Program ID** | 32 Bytes | `Pubkey` | Address of the program being invoked. |
| **Account Specs**| 2 Bytes | `u16` | Number of accounts provided to this instruction. |
| **Accounts** | 33 * N Bytes| `Array` | List of `[1 byte flags, 32 byte pubkey]` per account. |
| **Data Length** | 2 Bytes | `u16` | Size of the instruction data payload (arguments). |
| **Data Payload** | Varies | `Bytes` | **The raw Anchor arguments (see below).** |

---

## 3. Anchor Program Data Layout

For programs using the **Anchor Framework**, the `Data Payload` is standardized to facilitate function routing and argument parsing.

| Byte Range | Component | Size | Description |
| :--- | :--- | :--- | :--- |
| **0 - 7** | **Discriminator** | 8 Bytes | SHA256 hash of the function (e.g., `global:instruction_name`). |
| **8 - End** | **Arguments** | Varies | Serialized arguments in **Little Endian** (u64, u128, etc.). |

---

## Security Audit Checklist (Common Vulnerabilities)

### 1. Sysvar Spoofing

* **Risk:** Attacker passes a malicious account instead of the real Sysvar.
* **Fix:** Use `load_current_index_checked` or manually verify `account.key == sysvar::instructions::ID`.

### 2. CPI (Cross-Program Invocation) Masking

* **Risk:** A malicious contract calls your program via CPI to bypass introspection.
* **Fix:** Check `load_instruction_at_checked(current_index)` and verify the `program_id` matches your own Program ID. This ensures the call originated as a top-level instruction.

### 3. Discriminator/Data Collision

* **Risk:** Attacker calls a different function in your program that has similar data length.
* **Fix:** Hardcode the 8-byte discriminator and strictly enforce `data.len()` to the exact expected byte count.

### 4. Logic Gaps (Payback Missing)

* **Risk:** The loop finishes without finding the required instruction.
* **Fix:** Use a boolean flag (e.g., `payback_found`) and perform a final `require!(payback_found)` check after the loop completes.

### 5. Multi-Instruction Attacks

* **Risk:** Attacker includes multiple "Payback" instructions to manipulate accounting.
* **Fix:** Ensure the logic fails if more than one valid instruction is found within the search range.
