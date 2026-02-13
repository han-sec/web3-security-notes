# The `#[derive(Accounts)]` Macro

The `#[derive(Accounts)]` macro is used to define the **account context** for your instructions - essentially the list of accounts your instruction needs and how they should be validated.

## Basic Usage

```rust
#[derive(Accounts)]
pub struct Initialize<'info> {
    #[account(mut)]
    pub user: Signer<'info>,
    
    #[account(
        init,
        payer = user,
        space = 8 + 32 + 8
    )]
    pub user_account: Account<'info, UserAccount>,
    
    pub system_program: Program<'info, System>,
}
```

Then use it in your instruction handler:

```rust
#[program]
pub mod my_program {
    pub fn initialize(ctx: Context<Initialize>) -> Result<()> {
        // Access accounts via ctx.accounts
        let user_account = &mut ctx.accounts.user_account;
        user_account.authority = ctx.accounts.user.key();
        Ok(())
    }
}
```

## How It Works

**Account Types**
The struct fields use special wrapper types that tell Anchor how to treat each account:

- **`Signer<'info>`** - Must be a signer on the transaction
- **`Account<'info, T>`** - Deserializes into your custom account type `T`
- **`Program<'info, T>`** - Validates it's an executable program account
- **`AccountInfo<'info>`** - Raw account (no deserialization/validation)
- **`SystemAccount<'info>`** - Native system account
- **`UncheckedAccount<'info>`** - Explicitly unchecked (use carefully!)

**Validation Flow**
When an instruction is called:

1. Anchor receives raw account array from Solana runtime
2. Maps accounts to struct fields in order
3. Runs all `#[account(...)]` constraint checks
4. Deserializes accounts into appropriate types
5. Passes validated context to your handler

## The `'info` Lifetime

The `'info` lifetime ensures accounts can't outlive the instruction execution - preventing use-after-free bugs.

## Generic Contexts

You can make contexts generic for reusability:

```rust
#[derive(Accounts)]
pub struct Transfer<'info> {
    #[account(mut)]
    pub from: Account<'info, UserAccount>,
    
    #[account(mut)]
    pub to: Account<'info, UserAccount>,
    
    pub authority: Signer<'info>,
}
```

## Access Patterns

Inside your instruction handler:

```rust
pub fn my_instruction(ctx: Context<MyAccounts>, amount: u64) -> Result<()> {
    // Access individual accounts
    let user = &ctx.accounts.user;
    
    // Mutable access
    let user_account = &mut ctx.accounts.user_account;
    user_account.balance += amount;
    
    // Get account keys
    let user_key = ctx.accounts.user.key();
    
    Ok(())
}
```

## Nested Accounts

You can compose account contexts:

```rust
#[derive(Accounts)]
pub struct UpdateUser<'info> {
    #[account(mut)]
    pub user: Account<'info, UserAccount>,
    
    pub token_accounts: TransferAccounts<'info>,
}

#[derive(Accounts)]
pub struct TransferAccounts<'info> {
    pub token_program: Program<'info, Token>,
    
    #[account(mut)]
    pub from: Account<'info, TokenAccount>,
}
```

## Key Benefits

**Type Safety** - Can't mix up account order or types
**Automatic Validation** - All security checks generated at compile time
**Self-Documenting** - Clear intent of what accounts are needed
**Client Generation** - IDL automatically knows account structure for TypeScript
