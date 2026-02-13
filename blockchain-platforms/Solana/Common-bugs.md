# Common Solana Bugs

## Table of Contents

- [Account Initialization DoS](#account-initialization-dos)

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
