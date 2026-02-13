# Solana Account-Based Architecture

Unlike Ethereum, where smart contract state (like ERC20 balances) is stored in internal mappings, Solana uses an **account-based model**. Each piece of state lives in its own **account**, which is a discrete on-chain data container.

---

## 1. Accounts

* An **account** is the basic unit of storage on Solana.
* Each account has:

  * **Data**: raw bytes (`[u8]`) storing the state.
  * **Owner**: the program ID that is allowed to modify this account.
  * **Lamports**: Solana’s native tokens for rent and transaction fees.
* The runtime enforces **ownership**, ensuring that only the owning program can write to an account.

```
[User Account]
┌───────────────────────────┐
│ Owner: ProgramID           │
│ Lamports: 1_000_000        │
│ Data: [raw bytes]          │
└───────────────────────────┘
```

---

## 2. Program Derived Accounts (PDAs)

* **PDAs** are deterministic accounts derived from a program ID and a set of seeds.
* PDAs **do not have private keys**, so they cannot sign transactions themselves.
* Programs can **authorize PDAs to hold state**, ensuring only the program can mutate their data.
* Example: storing user balances in a vault program.

```
Vault Program ID
┌──────────────────────┐
│ PDA (User Balance)   │<-- only writable by Vault Program
│ Owner: Vault Program │
│ Data: { balance: 100 }│
└──────────────────────┘
```

---

## 3. Program ID

* Every program deployed on Solana has a unique **Program ID** (its public key).
* The Program ID is used as the **owner** for accounts it manages.
* Ownership enforces capability-based access control:

  * Only the program owning an account can deserialize and write its data.
  * Other programs or users cannot mutate the account directly.

```
Transaction Flow:

[User] --calls--> [Vault Program]
                     │
                     ▼
              [PDA / Account owned by Vault Program]
              Data: balance = 100
```

---

## 4. Example: Token Program

For a token program:

* Each user’s token balance is stored in a separate **token account**, often a PDA owned by the token program.
* To update a user’s balance:

  1. The program deserializes the account data (`[u8]`) into a Rust struct.
  2. Applies logic (e.g., increment, decrement).
  3. Serializes the struct back into the account’s bytes.
* This ensures all updates go **through the program**, similar to Ethereum contracts, but with **explicit account ownership**.

```
Token Program
┌─────────────────────────────┐
│ program logic               │
│ - transfer                  │
│ - mint                      │
└─────────────────────────────┘
          ▲
          │ owns
          │
┌───────────────┐    ┌───────────────┐
│ AliceAccount  │    │ BobAccount    │
│ Owner: Token  │    │ Owner: Token  │
│ Data: 1000    │    │ Data: 500     │
└───────────────┘    └───────────────┘
```

---

## 5. Key Takeaways

| Concept            | Ethereum ERC20         | Solana SPL Token / PDA       |
| ------------------ | ---------------------- | ---------------------------- |
| Balance storage    | Internal mapping       | Separate account per user    |
| Mutate permission  | Contract only          | Only owning program          |
| External access    | Call contract function | Must pass account to program |
| Ownership enforced | Implicit               | Explicit via `account.owner` |
| Serialization      | Handled by EVM         | Explicit (borsh / manual)    |

In short: **PDAs and accounts allow Solana programs to securely manage state**, enforce ownership, and maintain modular, deterministic on-chain storage, while keeping logic similar to Ethereum contracts.
