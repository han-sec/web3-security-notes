# Account Macro Parameters

The `#[account(...)]` macro accepts various constraints that tell Anchor how to validate accounts. Here are the main parameters:

## Ownership & Initialization

**`init`**

- Creates and initializes a new account
- Requires `payer` and `space` to be specified
- Automatically makes the account owned by your program

**`init_if_needed`**

- Initializes account if it doesn't exist, otherwise uses existing
- Use cautiously - can have security implications

**`payer = <account>`**

- Specifies who pays for account creation (rent)
- Used with `init`

**`space = <bytes>`**

- Defines account size in bytes
- Formula: `8 (discriminator) + account_data_size`

## Mutability

**`mut`**

- Marks account as mutable
- Required if instruction will modify the account
- Generates runtime check that account is writable

## Account Type Constraints

**`owner = <program_id>`**

- Validates the account is owned by specified program
- Can use `owner = token::ID` for SPL Token accounts

**`constraint = <expression>`**

- Custom validation logic
- Example: `constraint = user.authority == authority.key()`

**`has_one = <field>`**

- Validates that account's field matches another account's key
- Example: `has_one = authority` checks `account.authority == authority.key()`

## Seeds & PDAs

**`seeds = [...]`**

- Defines seeds for Program Derived Address (PDA)
- Used with `bump` to derive account address

**`bump`**

- The bump seed for PDA derivation
- Can store: `bump = user.bump` or let Anchor find it: `bump`

**`seeds::program = <program_id>`**

- Specifies which program to derive PDA from (defaults to current program)

## Token Accounts (SPL)

**`token::mint = <account>`**

- For token accounts, validates the mint address

**`token::authority = <account>`**

- Validates the token account's authority

**`associated_token::mint = <account>`**

- For Associated Token Accounts
- Used with `associated_token::authority`

## State Management

**`close = <account>`**

- Closes account and sends remaining lamports to specified account
- Prevents rent refund attacks

**`realloc = <size>`**

- Resizes an existing account
- Requires `realloc::payer` and `realloc::zero`

## Common Combinations

```rust
#[account(
    init,
    payer = user,
    space = 8 + 32 + 8,
    seeds = [b"user", user.key().as_ref()],
    bump
)]
pub user_account: Account<'info, UserAccount>,

#[account(
    mut,
    has_one = authority,
    constraint = user_account.amount > 0 @ ErrorCode::InsufficientFunds
)]
pub user_account: Account<'info, UserAccount>,
```
