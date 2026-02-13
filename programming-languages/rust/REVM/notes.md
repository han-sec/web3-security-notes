# REVM (Rust Ethereum Virtual Machine) – Quick Reference

`revm` is a high-performance Ethereum Virtual Machine implementation in Rust.  
It is **modular**, **configurable**, and **pluggable**, making it a common choice for execution in projects like Reth and Foundry.

## 1. Core Concepts

| Component | Description |
|-----------|-------------|
| **EVM** | The execution engine for Ethereum bytecode and transactions. |
| **Env** | Environment struct containing chain config, block, and transaction data. |
| **Database** | Trait abstraction for accessing Ethereum account/storage state. |
| **Primitives** | Core types like `Address`, `U256`, `TxEnv`, `BlockEnv`, `CfgEnv`. |
| **Inspectors** | Optional hooks for step-by-step execution tracing. |

## 2. Environment (`Env`)

`Env` contains three sub-envs:

```rust
Env {
    cfg: CfgEnv,     // Chain configuration & rules
    block: BlockEnv, // Block context (timestamp, basefee, coinbase)
    tx: TxEnv,       // Transaction context (caller, calldata, value, gas limit)
}
````

**Common Fields:**

```rust
env.cfg.chain_id        // U256: Chain ID (e.g., 1 for mainnet)
env.cfg.spec_id         // Hardfork spec (e.g., SpecId::MERGE)
env.block.number        // Current block number
env.block.timestamp     // Current block timestamp
env.block.basefee       // Base fee per gas
env.tx.caller           // Address of transaction sender
env.tx.to               // Optional<Address> recipient or contract
env.tx.data             // Bytes: calldata
env.tx.value            // U256: value sent
env.tx.gas_limit        // u64: gas limit
```

## 3. Database (`Database` Trait)

`Database` abstracts Ethereum state access.

**Built-in Implementations:**

* `EmptyDB` → Returns empty accounts unless manually populated.
* `StateBuilder` → Simple in-memory state builder for testing.
* Custom → Implement your own backend to pull state from RPC or local DB.

**Common Methods:**

```rust
db.basic(address)        // Get AccountInfo (balance, nonce, code hash)
db.code_by_hash(hash)    // Get code by code hash
db.storage(address, key) // Get storage value for given slot
```

## 4. Execution

**Most Common Method:**

```rust
evm.transact()
```

Runs a transaction in the configured environment against the given database.

**Other Methods:**

```rust
evm.inspect(inspector) // Run with a custom step-by-step inspector
evm.transact_ref()     // Same as transact but returns borrowed data
evm.commit()           // Commit state changes to the database
evm.reset()            // Reset EVM to an initial state
```

---

## 5. Typical Workflow

1. **Set up database**

   ```rust
   let mut db = StateBuilder::default()
       .account(
           addr,
           AccountInfo { balance: U256::from(1000), ..Default::default() }
       )
       .build();
   ```

2. **Create EVM and assign DB**

   ```rust
   let mut evm = EVM::new();
   evm.database(db);
   ```

3. **Configure environment**

   ```rust
   evm.env.cfg.chain_id = 1.into();
   evm.env.block.number = 17_000_000.into();
   evm.env.tx.caller = caller_addr;
   evm.env.tx.to = Some(contract_addr);
   evm.env.tx.data = calldata_bytes.into();
   evm.env.tx.gas_limit = 30_000_000;
   ```

4. **Execute**

   ```rust
   let result = evm.transact().unwrap();
   println!("Gas used: {:?}", result.gas_used);
   println!("Status: {:?}", result.result);
   ```

## 6. Inspecting Results

`ExecutionResult` contains:

```rust
result.gas_used     // Gas consumed
result.result       // Execution status & return data
result.logs         // Emitted logs
result.state        // State changes (before commit)
```

## 7. Example: Balance Query

```rust
use revm::{primitives::*, EVM};
use revm_database::StateBuilder;

fn main() {
    let address: Address = "0x0123456789abcdef0123456789abcdef01234567"
        .parse()
        .unwrap();

    let db = StateBuilder::default()
        .account(
            address,
            AccountInfo {
                balance: U256::from(1_000_000u64),
                ..Default::default()
            },
        )
        .build();

    let mut evm = EVM::new();
    evm.database(db);

    let info = evm.db().basic(address);
    println!("Balance: {}", info.balance);
}
```

## 8. References

* [revm GitHub](https://github.com/bluealloy/revm)
* [docs.rs – revm](https://docs.rs/revm/latest/revm/)
* [revm-database crate](https://docs.rs/revm-database)
