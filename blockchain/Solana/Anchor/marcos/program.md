# The `#[program]` Macro

The `#[program]` macro is where you define your program's **instruction handlers** - the actual business logic that gets executed when someone calls your program.

## Basic Structure

```rust
#[program]
pub mod my_program {
    use super::*;
    
    pub fn initialize(ctx: Context<Initialize>, amount: u64) -> Result<()> {
        let user_account = &mut ctx.accounts.user_account;
        user_account.authority = ctx.accounts.user.key();
        user_account.balance = amount;
        Ok(())
    }
    
    pub fn transfer(ctx: Context<Transfer>, amount: u64) -> Result<()> {
        let from = &mut ctx.accounts.from;
        let to = &mut ctx.accounts.to;
        
        require!(from.balance >= amount, ErrorCode::InsufficientFunds);
        
        from.balance -= amount;
        to.balance += amount;
        
        Ok(())
    }
}
```

## How It Works Under the Hood

**Entry Point Generation**
The `#[program]` macro generates the Solana program entry point for you. In raw Solana, you'd manually write:

```rust
entrypoint!(process_instruction);

pub fn process_instruction(
    program_id: &Pubkey,
    accounts: &[AccountInfo],
    instruction_data: &[u8],
) -> ProgramResult {
    // Manually parse instruction_data
    // Manually route to handlers
    // Manually deserialize accounts
}
```

Anchor handles all these

**Instruction Routing**
Each public function becomes an instruction with:

- A discriminator (first 8 bytes = hash of function name)
- Automatic serialization/deserialization of parameters
- Automatic account validation via the `Context<T>`

## Function Signature

Every instruction handler follows this pattern:

```rust
pub fn instruction_name(
    ctx: Context<AccountsStruct>,  // Required first parameter
    param1: Type1,                  // Optional parameters
    param2: Type2,
    // ...
) -> Result<()>  // Or Result<ReturnType>
```

**Required Components:**

- `ctx: Context<T>` - Must be first parameter
- `Result<()>` - Return type (or `Result<T>` for return values)

**The Context Type:**

```rust
pub struct Context<'a, 'b, 'c, 'info, T> {
    pub program_id: &'a Pubkey,
    pub accounts: &'b mut T,
    pub remaining_accounts: &'c [AccountInfo<'info>],
    pub bumps: BTreeMap<String, u8>,
}
```

## Instruction Parameters

You can pass additional data to instructions:

```rust
pub fn create_post(
    ctx: Context<CreatePost>,
    title: String,
    content: String,
    tags: Vec<String>,
) -> Result<()> {
    let post = &mut ctx.accounts.post;
    post.title = title;
    post.content = content;
    post.tags = tags;
    Ok(())
}
```

Anchor handles:

- Serialization of arguments in the client
- Deserialization in the program
- Type safety across the boundary

## Access Patterns

**Accounts Access:**

```rust
pub fn my_instruction(ctx: Context<MyAccounts>) -> Result<()> {
    // Immutable reference
    let user = &ctx.accounts.user;
    
    // Mutable reference
    let account = &mut ctx.accounts.user_account;
    account.balance += 100;
    
    // Get program ID
    let program_id = ctx.program_id;
    
    // Access remaining accounts (unchecked accounts passed)
    let remaining = ctx.remaining_accounts;
    
    // Get PDA bumps
    let bump = ctx.bumps.get("user_account").unwrap();
    
    Ok(())
}
```

## Return Values

You can return data from instructions:

```rust
pub fn calculate(ctx: Context<Calculate>, x: u64, y: u64) -> Result<u64> {
    Ok(x + y)
}
```

The return value is serialized and available in transaction logs.

## What Gets Generated

For each instruction function, Anchor generates:

1. **Discriminator** - 8-byte hash of `"global:function_name"`
2. **Wrapper function** - Handles deserialization and calls your handler
3. **Instruction builder** - Client-side helper for building transactions
4. **TypeScript types** - Automatic TS client generation

## Example: What Anchor Does For You

When you write:

```rust
#[program]
pub mod my_program {
    pub fn increment(ctx: Context<Increment>) -> Result<()> {
        ctx.accounts.counter.count += 1;
        Ok(())
    }
}
```

Anchor generates (conceptually):

```rust
// Entry point
fn process_instruction(
    program_id: &Pubkey,
    accounts: &[AccountInfo],
    data: &[u8],
) -> ProgramResult {
    // Parse discriminator
    let discriminator = &data[0..8];
    
    if discriminator == INCREMENT_DISCRIMINATOR {
        // Validate and deserialize accounts
        let ctx = Increment::try_accounts(...)?;
        // Call your handler
        increment(ctx)?;
        // Exit accounts (persist changes)
        ctx.exit(program_id)?;
    }
    
    Ok(())
}
```

## Multiple Instructions

You can have as many instruction handlers as you need:

```rust
#[program]
pub mod my_program {
    pub fn initialize(ctx: Context<Initialize>) -> Result<()> { ... }
    pub fn update(ctx: Context<Update>) -> Result<()> { ... }
    pub fn delete(ctx: Context<Delete>) -> Result<()> { ... }
    pub fn transfer(ctx: Context<Transfer>, amount: u64) -> Result<()> { ... }
}
```

Each becomes a separate callable instruction with its own discriminator.

## Key Takeaways

The `#[program]` macro:

- **Generates the entire entry point** - No manual instruction routing
- **Handles serialization** - Automatic parameter and account handling  
- **Creates discriminators** - Unique identifiers for each instruction
- **Builds client code** - TypeScript functions generated automatically
- **Type safety** - Compile-time guarantees across client/program boundary
