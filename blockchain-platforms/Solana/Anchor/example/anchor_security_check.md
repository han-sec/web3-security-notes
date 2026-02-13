# Anchor Security Checks: Code Mapping Guide

This document maps specific sections of Anchor's expanded code to the security checks they implement. The following write up maps the anchor validation based on an [anchor program sample code](./anchor_sample_code.rs)

---

## 1. Program ID Verification

**Check:** Ensures the called program matches the declared program ID

```rust
fn try_entry<'info>(
    program_id: &Pubkey,
    accounts: &'info [AccountInfo<'info>],
    data: &[u8],
) -> anchor_lang::Result<()> {
    if *program_id != ID {
        return Err(anchor_lang::error::ErrorCode::DeclaredProgramIdMismatch.into());
    }
    dispatch(program_id, accounts, data)
}
```

**What it prevents:** Program impersonation attacks  
**Why it matters:** Without this, malicious programs could pretend to be your program

---

## 2. Instruction Discriminator Validation

**Check:** Validates instruction type via 8-byte discriminator and map discriminator to the function call

```rust
fn dispatch<'info>(
    program_id: &Pubkey,
    accounts: &'info [AccountInfo<'info>],
    data: &[u8],
) -> anchor_lang::Result<()> {
    if data.starts_with(instruction::InitializeMint::DISCRIMINATOR) {
        return __private::__global::initialize_mint(
            program_id,
            accounts,
            &data[instruction::InitializeMint::DISCRIMINATOR.len()..],
        );
    }
    if data.starts_with(instruction::MintTokens::DISCRIMINATOR) {
        return __private::__global::mint_tokens(
            program_id,
            accounts,
            &data[instruction::MintTokens::DISCRIMINATOR.len()..],
        );
    }
    // ... more instructions
}
```

**Discriminator Examples (Lines 2609-2684):**

```rust
impl anchor_lang::Discriminator for InitializeMint {
    const DISCRIMINATOR: &'static [u8] = &[209, 42, 195, 4, 129, 85, 209, 44];
}

impl anchor_lang::Discriminator for MintTokens {
    const DISCRIMINATOR: &'static [u8] = &[59, 132, 24, 246, 122, 39, 8, 243];
}

impl anchor_lang::Discriminator for TransferTokens {
    const DISCRIMINATOR: &'static [u8] = &[54, 180, 238, 175, 74, 85, 126, 188];
}
```

**What it prevents:** Wrong instruction execution, instruction confusion attacks  
**Why it matters:** Ensures only the intended instruction logic is executed

---

## 3. Instruction Deserialization Check

**Check:** Validates instruction data format

```rust
// In initialize_mint handler
let ix = instruction::InitializeMint::deserialize(&mut &__ix_data[..])
    .map_err(|_| {
        anchor_lang::error::ErrorCode::InstructionDidNotDeserialize
    })?;
```

**What it prevents:** Malformed instruction data, buffer overflow attacks  
**Why it matters:** Ensures instruction data matches expected format before processing

---

## 4. Account Count Validation

**Check:** Ensures sufficient accounts provided, accounts array is not empty

```rust
fn try_accounts(
    __program_id: &anchor_lang::solana_program::pubkey::Pubkey,
    __accounts: &mut &'info [AccountInfo<'info>],
    __ix_data: &[u8],
    __bumps: &mut InitializeMintBumps,
    __reallocs: &mut std::collections::BTreeSet<Pubkey>,
) -> anchor_lang::Result<Self> {
    if __accounts.is_empty() {
        return Err(anchor_lang::error::ErrorCode::AccountNotEnoughKeys.into());
    }
    // ... continue validation
}
```

**What it prevents:** Missing account attacks  
**Why it matters:** Programs crash or behave unexpectedly without required accounts

---

## 5. Signer Verification

**Check:** Validates that required accounts signed the transaction

```rust
// For payer account
let payer: Signer = anchor_lang::Accounts::try_accounts(
        __program_id,
        __accounts,
        __ix_data,
        __bumps,
        __reallocs,
    )
    .map_err(|e| e.with_account_name("payer"))?;

// For authority account in MintTokens
let authority: Signer = anchor_lang::Accounts::try_accounts(
        __program_id,
        __accounts,
        __ix_data,
        __bumps,
        __reallocs,
    )
    .map_err(|e| e.with_account_name("authority"))?;
```

**What it prevents:** Unauthorized transactions  
**Why it matters:** Only the rightful owner can authorize actions on their accounts

---

## 6. Account Type Validation

**Check:** Ensures account data matches expected type (via discriminator)

```rust
// Validating mint account
let pa: anchor_lang::accounts::account::Account<Mint> = 
    match anchor_lang::accounts::account::Account::try_from_unchecked(&mint) {
        Ok(val) => val,
        Err(e) => return Err(e.with_account_name("mint")),
    };

// Validating token account in MintTokens
let token_account: anchor_lang::accounts::account::Account<TokenAccount> = 
    anchor_lang::Accounts::try_accounts(
        __program_id,
        __accounts,
        __ix_data,
        __bumps,
        __reallocs,
    )
    .map_err(|e| e.with_account_name("token_account"))?;
```

**What it prevents:** Type confusion attacks, using wrong account types  
**Why it matters:** Prevents reading a TokenAccount as a Mint, which could leak sensitive data

---

## 7. Account Ownership Verification

**Check:** Validates account is owned by the correct program

```rust
let owner_program = AsRef::<AccountInfo>::as_ref(&mint).owner;
if !false
    || owner_program == &anchor_lang::solana_program::system_program::ID
{
    // Account is uninitialized or owned by system program
    // Proceed with initialization...
}
```

**Note:** The `Account<T>` type wrapper automatically verifies ownership. When you use:

```rust
pub mint: Account<'info, Mint>,
```

Anchor automatically checks that the mint account is owned by the Token Program.

**What it prevents:** Account substitution attacks using accounts from wrong programs  
**Why it matters:** Ensures you're operating on accounts controlled by the expected program

---

## 8. Mutability (Writable) Check

**Check:** Ensures accounts marked `mut` are writable

```rust
if !AsRef::<AccountInfo>::as_ref(&mint).is_writable {
    return Err(
        anchor_lang::error::Error::from(
                anchor_lang::error::ErrorCode::ConstraintMut,
            )
            .with_account_name("mint"),
    );
}

if !AsRef::<AccountInfo>::as_ref(&token_account).is_writable {
    return Err(
        anchor_lang::error::Error::from(
                anchor_lang::error::ErrorCode::ConstraintMut,
            )
            .with_account_name("token_account"),
    );
}
```

**What it prevents:** Unauthorized state modifications  
**Why it matters:** Read-only accounts can't be modified, protecting against unintended changes

---

## 9. Program Account Validation

**Check:** Verifies program accounts match expected program IDs

```rust
// System Program validation
let system_program: anchor_lang::accounts::program::Program<System> = 
    anchor_lang::Accounts::try_accounts(
        __program_id,
        __accounts,
        __ix_data,
        __bumps,
        __reallocs,
    )
    .map_err(|e| e.with_account_name("system_program"))?;

// Token Program validation
let token_program: anchor_lang::accounts::program::Program<Token> = 
    anchor_lang::Accounts::try_accounts(
        __program_id,
        __accounts,
        __ix_data,
        __bumps,
        __reallocs,
    )
    .map_err(|e| e.with_account_name("token_program"))?;
```

**What it prevents:** Using fake/malicious program accounts  
**Why it matters:** Ensures CPIs (Cross-Program Invocations) call the legitimate programs

---

## 10. Account Initialization Checks

**Check:** Complex initialization validation including:

- Zero lamports check
- Space calculation
- Rent exemption calculation
- Account creation via CPI
- Lamport transfer if needed
- Allocate instruction
- Assign owner

```rust
let __current_lamports = mint.lamports();
if __current_lamports == 0 {
    // New account - create it
    let space = ::anchor_spl::token::Mint::LEN;
    let lamports = __anchor_rent.minimum_balance(space);
    let cpi_accounts = anchor_lang::system_program::CreateAccount {
        from: payer.to_account_info(),
        to: mint.to_account_info(),
    };
    let cpi_context = anchor_lang::context::CpiContext::new(
        system_program.to_account_info(),
        cpi_accounts,
    );
    anchor_lang::system_program::create_account(
        cpi_context.with_signer(&[]),
        lamports,
        space as u64,
        &token_program.key(),
    )?;
} else {
    // Account exists - validate and fund if needed
    if payer.key() == mint.key() {
        return Err(/* TryingToInitPayerAsProgramAccount */);
    }
    
    let required_lamports = __anchor_rent
        .minimum_balance(::anchor_spl::token::Mint::LEN)
        .max(1)
        .saturating_sub(__current_lamports);
        
    if required_lamports > 0 {
        // Transfer additional lamports for rent exemption
        let cpi_accounts = anchor_lang::system_program::Transfer {
            from: payer.to_account_info(),
            to: mint.to_account_info(),
        };
        anchor_lang::system_program::transfer(cpi_context, required_lamports)?;
    }
}
```

**What it prevents:**

- Reinitialization attacks (using already initialized accounts). Transaction will revert, unless using the `init_if_need`
- Rent violations (accounts without enough SOL getting garbage collected)
- Payer confusion (trying to init payer as program account)

**Why it matters:** Ensures accounts are properly set up and funded before use

---

## 11. Rent Exemption Validation

**Check:** Ensures accounts have sufficient lamports for rent exemption

```rust
// Get rent sysvar
let __anchor_rent = Rent::get()?;

// Calculate minimum balance for rent exemption
let lamports = __anchor_rent.minimum_balance(space);

// For existing accounts, check if more lamports needed
let required_lamports = __anchor_rent
    .minimum_balance(::anchor_spl::token::Mint::LEN)
    .max(1)
    .saturating_sub(__current_lamports);
```

**What it prevents:** Account deletion by rent collection  
**Why it matters:** Accounts without rent exemption get deleted, losing all data

---

## 12. Constraint Validation: has_one

**Check:** Validates foreign key relationships between accounts

```rust
{
    let my_key = idl.authority;
    let target_key = authority.key();
    if my_key != target_key {
        return Err(
            anchor_lang::error::Error::from(
                    anchor_lang::error::ErrorCode::ConstraintHasOne,
                )
                .with_account_name("idl")
                .with_pubkeys((my_key, target_key)),
        );
    }
}
```

**What it prevents:** Using unauthorized accounts for related operations  
**Why it matters:** Enforces that only the correct authority can modify associated accounts

---

## 13. Constraint Validation: Custom Constraints

**Check:** Validates custom business logic constraints

```rust
// Example: Authority cannot be erased
if !(authority.key != &ERASED_AUTHORITY) {
    return Err(
        anchor_lang::error::Error::from(
                anchor_lang::error::ErrorCode::ConstraintRaw,
            )
            .with_account_name("authority"),
    );
}
```

**What it prevents:** Violating custom business rules  
**Why it matters:** Enforces application-specific security requirements

---

## 14. Account Exit/Persistence

**Check:** Saves modified account state back to Solana

```rust
// At end of each instruction handler
__accounts.exit(__program_id)?;

// Exit implementation for MintTokens
fn exit(
    &self,
    program_id: &anchor_lang::solana_program::pubkey::Pubkey,
) -> anchor_lang::Result<()> {
    anchor_lang::AccountsExit::exit(&self.mint, program_id)
        .map_err(|e| e.with_account_name("mint"))?;
    anchor_lang::AccountsExit::exit(&self.token_account, program_id)
        .map_err(|e| e.with_account_name("token_account"))?;
    Ok(())
}
```

**What it prevents:** Lost state changes  
**Why it matters:** Without this, account modifications wouldn't be persisted

---

## 15. Error Context Enhancement

**Check:** Adds context to errors for better debugging

```rust
// Top-level error logging
pub fn entry<'info>(
    program_id: &Pubkey,
    accounts: &'info [AccountInfo<'info>],
    data: &[u8],
) -> anchor_lang::solana_program::entrypoint::ProgramResult {
    try_entry(program_id, accounts, data)
        .map_err(|e| {
            e.log();  // Log the error
            e.into()
        })
}

// Adding account name to errors
.map_err(|e| e.with_account_name("payer"))?;
```

**What it prevents:** Hard-to-debug errors  
**Why it matters:** Developers can quickly identify which account/check failed

---
