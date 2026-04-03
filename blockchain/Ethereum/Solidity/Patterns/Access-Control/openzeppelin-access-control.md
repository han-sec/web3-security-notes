# OpenZeppelin Access Control Patterns

## The Problem

Smart contracts need to restrict who can call certain functions. "Only the admin can pause" or "only whitelisted users can mint." OZ provides three levels of complexity.

---

## 1. Ownable — One Owner, One Permission Level

```solidity
contract MyContract is Ownable {
    function pause() external onlyOwner { ... }
    function setFee() external onlyOwner { ... }
}
```

- Single `owner` address stored in contract
- One modifier: `onlyOwner`
- Owner can `transferOwnership(newOwner)` or `renounceOwnership()`

**Variants:**

| Contract | Transfer Mechanism | When to Use |
|---|---|---|
| `Ownable` | Direct (one tx) | Low-stakes, simple contracts |
| `Ownable2Step` | Propose + accept (two tx) | Prevents typo = permanent loss |

**Limitation:** All-or-nothing. Owner can do everything. No granularity.

---

## 2. AccessControl — Multiple Roles, Multiple Holders

```solidity
contract MyContract is AccessControl {
    bytes32 public constant MINTER_ROLE = keccak256("MINTER_ROLE");
    bytes32 public constant PAUSER_ROLE = keccak256("PAUSER_ROLE");

    function mint() external onlyRole(MINTER_ROLE) { ... }
    function pause() external onlyRole(PAUSER_ROLE) { ... }
}
```

Internal storage is a nested mapping:
```solidity
mapping(bytes32 role => mapping(address => bool)) private _roles;
```

**Key concepts:**

- Multiple roles, each identified by a `bytes32`
- Multiple addresses can hold the same role
- Each role has an `adminRole` (defaults to `DEFAULT_ADMIN_ROLE`)
- Only the admin of a role can grant/revoke it

```
DEFAULT_ADMIN_ROLE (admin of all roles by default)
  ├── can grant/revoke MINTER_ROLE
  ├── can grant/revoke PAUSER_ROLE
  └── can grant/revoke itself
```

**Variants:**

| Contract | Extra Feature | When to Use |
|---|---|---|
| `AccessControl` | Base implementation | Most cases |
| `AccessControlEnumerable` | Can list all holders of a role on-chain | Need on-chain enumeration (rare) |
| `AccessControlDefaultAdminRules` | 2-step + time delay for admin transfer | High-value protocols |

---

## 3. AccessControlDefaultAdminRules — Safe Admin Transfer

The `DEFAULT_ADMIN_ROLE` is the most powerful role — it controls all other roles. Transferring it carelessly is dangerous.

```solidity
contract MyContract is AccessControlDefaultAdminRules {
    constructor() AccessControlDefaultAdminRules(3 days, msg.sender) { }
}
```

Transfer process:
```
Day 1: admin calls beginDefaultAdminTransfer(newAdmin)
Day 4: newAdmin calls acceptDefaultAdminTransfer()  // after 3-day delay
```

- Cannot renounce without going through 2-step process
- Configurable delay (e.g., 3 days)
- Prevents impulsive or accidental admin transfers

---

## Roles vs Whitelists — When to Use What

**AccessControl roles** are for protocol operators:
```
PAUSER_ROLE         → ops team (2-3 addresses)
RESOLVER_ROLE       → oracle adapter (1 address)
FINALISER_ROLE      → oracle adapter (1 address)
```

**Custom mappings** are for user-level access gates:
```solidity
mapping(address => bool) public whitelistedCreators;
```

**Why separate them:**

1. **Principle of least privilege** — roles carry implicit power (adminRole can grant/revoke). A mapping gates exactly one thing with no side effects.

2. **Different lifecycle** — roles change rarely (multisig tx). Whitelists may change more often, managed by a less privileged operator.

3. **Clarity** — `onlyRole(PAUSER_ROLE)` clearly means admin operation. `require(whitelisted[msg.sender])` clearly means user access gate.

**Rule of thumb:**
```
Small set, high privilege, manages the protocol  → AccessControl role
Larger set, single purpose, uses the protocol    → mapping whitelist
Very large set, gas-sensitive                    → Merkle proof
```

---

## Common Patterns in Production

```
Uniswap:     governance (timelock) for admin, mappings for fee tier whitelist
Aave:        AccessControl for admin roles, mappings for asset whitelist
Compound:    single admin (timelock), mappings for market whitelist
Polymarket:  single admin (Auth mixin), no whitelisting (team creates all markets)
```

---

## Quick Reference

```solidity
// Ownable
owner()                              // get owner
transferOwnership(newOwner)          // transfer (direct or 2-step)
renounceOwnership()                  // give up ownership

// AccessControl
hasRole(role, account)               // check
grantRole(role, account)             // grant (only admin of role)
revokeRole(role, account)            // revoke (only admin of role)
renounceRole(role, account)          // self-revoke
getRoleAdmin(role)                   // which role is admin of this role

// AccessControlDefaultAdminRules
beginDefaultAdminTransfer(newAdmin)  // step 1
acceptDefaultAdminTransfer()         // step 2 (after delay)
defaultAdminDelay()                  // get the configured delay
```
