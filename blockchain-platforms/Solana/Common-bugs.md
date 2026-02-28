# Common Solana Bugs

## Table of Contents

- [Account Initialization DoS](#account-initialization-dos)
- [Rent is not linear](#rent-is-not-linear)

---

## Account Initialization DoS

### The Problem

Using `system_program::create_account` has a DoS vulnerability:

- If an account already has lamports, the instruction **fails**
- Attackers can pre-fund PDA addresses to permanently block initialization

### Solution: Use `init_if_needed`

```rust
#[derive(Accounts)]
pub struct InitAccount<'info> {
    #[account(
        init_if_needed,
        payer = user,
        space = 8 + MyAccount::LEN,
        seeds = [b"myaccount", user.key().as_ref()],
        bump
    )]
    pub my_account: Account<'info, MyAccount>,

    #[account(mut)]
    pub user: Signer<'info>,
    pub system_program: Program<'info, System>,
}
```

### Anchor Init Attributes

| Attribute | Behavior |
|-----------|----------|
| `init` | Reverts if account exists. Use for strict one-time initialization. |
| `init_if_needed` | Idempotent. Reuses existing account if it has lamports. Safe against DoS. |

### Key Points

- Always use `init_if_needed` for PDAs that could be front-run
- Validate `owner`, `seeds`, and `discriminator` for security
- Never assume an account has zero lamports

<a id="rent-is-not-linear"> ## Rent is not linear</a>

### Summary

On Solana, **account rent is not linear with account size**. Developers often assume that doubling the account size doubles the rent, but this is incorrect because **rent includes a fixed base cost** in addition to a per-byte cost.

This leads to **overcharging users**, incorrect lamport accounting, transaction failures, and potential DoS vectors when resizing or reallocating accounts.

---

### Core Concept

Rent follows an **affine function**, not a linear one:

```
rent(size) = BASE + (size × RATE)
```

Where:

- `BASE` = fixed minimum rent (even for 0 bytes)
- `RATE` = rent cost per byte

This means:

```
rent(20) ≠ 2 × rent(10)
```

Because `BASE` would be charged twice.

---

### Common Developer Mistake

When resizing an account from `old_size` to `new_size = old_size + diff`, some implementations incorrectly compute:

```
rent(old_size) + rent(diff)
```

This is **wrong**, because it charges the base rent twice.

Correct calculation:

```
rent(new_size)
```

or equivalently:

```
rent(new_size) - rent(old_size)
```

---

### Example

Assume:

```
BASE = 1000 lamports
RATE = 10 lamports / byte
```

Account grows from 10 → 15 bytes.

**Wrong approach:**

```
rent(10) + rent(5)
= (1000 + 100) + (1000 + 50)
= 2150
```

**Correct approach:**

```
rent(15)
= 1000 + 150
= 1150
```

➡ **Bug: Overcharging 1000 lamports**

---

### Impact

This bug can lead to:

- Overcharging users
- Failed reallocations due to insufficient lamports
- Unexpected transaction reverts
- DoS vectors through forced realloc failures
- Broken accounting invariants

---

### Where This Commonly Appears

- PDA resizing
- Dynamic account growth
- Metadata extensions
- Account migrations
- Realloc instructions
- Program upgrades

---

### Secure Coding Pattern

Always compute rent using **total final size**:

```rust
let old_rent = Rent::get()?.minimum_balance(old_size);
let new_rent = Rent::get()?.minimum_balance(new_size);
let lamports_required = new_rent - old_rent;
```

Never compute rent based only on the size delta.

---

### Auditor Checklist

When auditing Solana programs, flag any logic that:

- Calculates rent incrementally
- Uses `rent(diff)`
- Adds two rent values together

Correct logic must always use:

```
rent(new_size)
```
