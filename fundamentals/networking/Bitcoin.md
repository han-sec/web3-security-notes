# 🧱 What's Inside a Bitcoin Block?

Every \~10 minutes, a new Bitcoin block is mined — a heartbeat that secures the network, confirms transactions, and incentivizes miners. But what exactly is inside a block?

Let’s crack it open and explore the **block header**, the **transaction body**, and the special mechanics like `OP_RETURN`, `coinbase transactions`, and `SegWit commitments`.

---

## 🔗 The Block Header — 80 Bytes of Proof

The **block header** is a compact summary of the entire block. At just **80 bytes**, it contains everything required for:

* **Linking blocks together** (the blockchain)
* **Enabling mining** (proof-of-work)
* **Ensuring validity** through hashes and timestamps

### 🧩 Block Header Structure

| Field                 | Size     | Description                                                 |
| --------------------- | -------- | ----------------------------------------------------------- |
| `version`             | 4 bytes  | Signals which validation rules are used (e.g., BIP9/BIP141) |
| `previous_block_hash` | 32 bytes | Hash of the previous block (creates the chain)              |
| `merkle_root`         | 32 bytes | Root of Merkle tree of all transactions                     |
| `timestamp`           | 4 bytes  | When this block was mined                                   |
| `bits`                | 4 bytes  | Encodes the current mining difficulty                       |
| `nonce`               | 4 bytes  | Adjusted by miners to find a valid hash                     |

> 🧠 **Fun Fact:** Miners cycle the `nonce` and `extraNonce` in the coinbase scriptSig to find a block hash below the target defined by `bits`.

---

## 📦 The Block Body — Where the Transactions Live

The **block body** contains:

* A **varint** indicating the number of transactions
* A list of **transactions** (including one special coinbase transaction)

Each transaction consists of:

* **Inputs**: Which UTXOs are being spent
* **Outputs**: Where the value goes
* **Witnesses**: Signatures and scripts (if SegWit is used)

---

## 🔁 The Coinbase Transaction — Minting New Bitcoin

The **first transaction** in a block is always the **coinbase transaction**, created by the miner.

### 💰 Characteristics

* Has **no inputs**
* **Creates new BTC** (subsidy + transaction fees)
* Often includes **metadata** like:

  * Block height
  * Miner identifier
  * SegWit commitment

### 🧱 Example Structure (Rust-style Pseudocode)

```rust
BitcoinTransaction {
    version: 2,
    inputs: [
        TxIn {
            previous_output: null,
            script_sig: "<block height><extra nonce><miner tag>",
            sequence: 0xffffffff
        }
    ],
    outputs: [
        TxOut {
            value: <block reward + fees>,
            script_pubkey: "<P2PKH or P2WPKH>"
        },
        TxOut {
            value: 0,
            script_pubkey: "OP_RETURN <witness commitment or metadata>"
        }
    ]
}
```

> 🧠 **Multiple Outputs?** Yes, a coinbase transaction can have multiple outputs for:
>
> * Mining pool reward distribution
> * `OP_RETURN` metadata
> * Non-BTC incentives

---

## 🔄 Regular Transactions — Spending UTXOs

### 🔹 Inputs

Each **input** spends a previously unspent output (UTXO). It includes:

* **Outpoint**: Transaction ID + output index
* **scriptSig**: Unlocking script (legacy)
* **sequence**: For time-locked transactions
* **witness**: Signatures (for SegWit)

### 🔸 Outputs

Each **output** defines where the BTC is sent and how it can be spent.

```rust
TxOut {
    value: Amount,
    script_pubkey: ScriptBuf
}
```

### Common Output Types

| Output Type    | Format                                                      | Description                            |
| -------------- | ----------------------------------------------------------- | -------------------------------------- |
| **P2PKH**      | `OP_DUP OP_HASH160 <pubKeyHash> OP_EQUALVERIFY OP_CHECKSIG` | Most common legacy type                |
| **P2WPKH**     | `0x00 <20-byte pubKeyHash>`                                 | Native SegWit (starts with `bc1q`)     |
| **P2SH**       | `OP_HASH160 <scriptHash> OP_EQUAL`                          | Wraps complex scripts                  |
| **OP\_RETURN** | `OP_RETURN <data>`                                          | Carries metadata, provably unspendable |

---

## 🧨 OP\_RETURN — Embedding Data in the Blockchain

`OP_RETURN` creates an **unspendable output**. It is widely used to:

* Store **metadata** (e.g., timestamps, messages)
* Commit to off-chain data (like in Ordinals, Stacks, Counterparty)
* Record **SegWit witness commitments**

### 🔖 Format

```
<OP_RETURN (0x6a)> <PUSHDATA opcode> <data>
```

### 🧠 Example — Hello World

```json
TxOut {
  value: 0,
  script_pubkey: "6a1048656c6c6f20576f726c64"
}
```

> That hex decodes to `"Hello World"`.

---

## 🧬 SegWit & Witness Commitment (BIP141)

**SegWit** (Segregated Witness) was introduced to:

* Solve **transaction malleability**
* Increase **block capacity**
* Separate **signatures (witnesses)** from main tx data

### 📛 Witness Commitment in OP\_RETURN

In **SegWit-enabled blocks**, the **witness merkle root** is committed via a special `OP_RETURN` output in the coinbase transaction.

### 🔖 Format

```
OP_RETURN 6a24aa21a9ed<witness_merkle_root>
```

* `aa21a9ed` = Magic prefix (4 bytes)
* `<witness_merkle_root>` = SHA256(root of witness data + nonce)

> 🛡 Ensures full nodes can validate witness data even though it's stored outside the main tx body.

---

## ⚖️ Block Weight Math (BIP141 Rules)

| Data Type        | Weight Units per Byte |
| ---------------- | --------------------- |
| Non-witness data | 4                     |
| Witness data     | 1                     |

* Max block weight: **4,000,000 weight units**
* Practical block size: \~2MB-4MB depending on usage

---

## 🧠 TL;DR — Anatomy of a Bitcoin Block

```
Block
├── Header (80 bytes)
│   ├── Version
│   ├── Previous Block Hash
│   ├── Merkle Root
│   ├── Timestamp
│   ├── Bits (Difficulty)
│   └── Nonce
└── Body
    ├── Transaction Count
    └── Transactions
        ├── Coinbase Tx (minted BTC, OP_RETURN, metadata)
        ├── Standard Txs (P2PKH, P2WPKH, etc.)
        └── Optional: SegWit Witnesses
```

---

## 👁 Final Thoughts

A Bitcoin block is far more than a bundle of transactions. It’s a **self-contained cryptographic record**, enforcing consensus and incentivizing honest behavior.

Understanding the structure of a block gives you the tools to:

* **Decode raw block data**
* **Trace Bitcoin flows**
* **Build indexers, explorers, or mempool monitors**
* **Embed or extract metadata safely**
