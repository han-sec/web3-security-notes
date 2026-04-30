# OpenZeppelin Access Control Patterns

## The Problem

Smart contracts need to restrict who can call certain functions. OZ provides a hierarchy of access control contracts with increasing complexity and safety guarantees.

---

## Overview: Which Contract to Use

| Contract | Roles | Transfer Safety | Admin Delay | Upgradeable? |
| --- | --- | --- | --- | --- |
| `Ownable` | 1 (owner) | Direct (1-tx) | No | `OwnableUpgradeable` |
| `Ownable2Step` | 1 (owner) | 2-step (accept required) | No | `Ownable2StepUpgradeable` |
| `AccessControl` | Many | Direct per-role | No | `AccessControlUpgradeable` |
| `AccessControlDefaultAdminRules` | Many | 2-step + delay for admin | Yes (configurable) | `AccessControlDefaultAdminRulesUpgradeable` |

---

## 1. Ownable

### How It Works

```solidity
contract MyContract is Ownable {
    constructor() Ownable(msg.sender) {}

    function pause() external onlyOwner { ... }
    function setFee(uint256 fee) external onlyOwner { ... }
}
```

### Internal Storage

```solidity
// Ownable.sol (OZ v5)
address private _owner;
```

A single slot storing the owner address.

### Key Functions

```solidity
owner()                          // returns current owner
transferOwnership(address newOwner) // sets _owner = newOwner immediately
renounceOwnership()              // sets _owner = address(0), permanently
```

`transferOwnership` emits `OwnershipTransferred(oldOwner, newOwner)`.

### The `onlyOwner` Modifier

```solidity
modifier onlyOwner() {
    if (owner() != msg.sender) revert OwnableUnauthorizedAccount(msg.sender);
    _;
}
```

Single SLOAD to read `_owner`, compare to `msg.sender`, revert or continue.

### Limitation

One address controls everything. A single compromised or typo'd `transferOwnership` call permanently transfers all control. No granularity across functions.

---

## 2. Ownable2Step

### Two-Step Transfer Mechanism

Adds a two-transaction transfer to prevent permanent loss from a typo'd address.

```solidity
contract MyContract is Ownable2Step {
    constructor() Ownable(msg.sender) {}
}
```

### Ownable2Step Storage

```solidity
// Ownable2Step.sol
address private _owner;           // inherited from Ownable
address private _pendingOwner;    // new: holds the proposed owner
```

### Transfer Flow

```
1. Current owner calls: transferOwnership(newOwner)
      → stores newOwner in _pendingOwner
      → emits OwnershipTransferStarted(currentOwner, newOwner)
      → _owner is NOT changed yet

2. newOwner calls: acceptOwnership()
      → verifies msg.sender == _pendingOwner
      → sets _owner = _pendingOwner
      → clears _pendingOwner = address(0)
      → emits OwnershipTransferred
```

If the new owner address was wrong, step 2 simply never happens and the current owner can start over.

### Ownable2Step Functions

```solidity
owner()                             // current owner
pendingOwner()                      // proposed new owner (or address(0))
transferOwnership(address newOwner) // starts transfer, sets pendingOwner
acceptOwnership()                   // completes transfer (called by pendingOwner)
renounceOwnership()                 // immediate, no 2-step protection here
```

> **Note:** `renounceOwnership()` is still direct (one-tx) in Ownable2Step. It is NOT protected by the 2-step mechanism. Be aware that a compromised owner key can still renounce immediately.

---

## 3. AccessControl

### Role-Based Model

Multiple roles, each identified by a `bytes32` constant, each with multiple holders. Any address can hold any number of roles. Each role has a designated "admin role" that controls who can grant/revoke it.

```solidity
contract MyContract is AccessControl {
    bytes32 public constant MINTER_ROLE = keccak256("MINTER_ROLE");
    bytes32 public constant PAUSER_ROLE = keccak256("PAUSER_ROLE");

    constructor() {
        _grantRole(DEFAULT_ADMIN_ROLE, msg.sender);
    }

    function mint(address to) external onlyRole(MINTER_ROLE) { ... }
    function pause() external onlyRole(PAUSER_ROLE) { ... }
}
```

### AccessControl Storage

```solidity
struct RoleData {
    mapping(address account => bool) hasRole;
    bytes32 adminRole;
}

mapping(bytes32 role => RoleData) private _roles;
```

Two nested mappings: `_roles[role].hasRole[account]` for membership, and `_roles[role].adminRole` for who manages that role.

### DEFAULT_ADMIN_ROLE

```solidity
bytes32 public constant DEFAULT_ADMIN_ROLE = 0x00;
```

It's `bytes32(0)`. By default, it is its own admin, meaning the `DEFAULT_ADMIN_ROLE` holder can grant/revoke any role (including itself). It must be explicitly granted in the constructor.

### Role Hierarchy (Default)

```
DEFAULT_ADMIN_ROLE (adminRole = itself)
  ├── can grant/revoke MINTER_ROLE
  ├── can grant/revoke PAUSER_ROLE
  └── can grant/revoke DEFAULT_ADMIN_ROLE
```

You can change which role is admin of another:

```solidity
// Only MINTER_ADMIN_ROLE can grant/revoke MINTER_ROLE
_setRoleAdmin(MINTER_ROLE, MINTER_ADMIN_ROLE);
```

### AccessControl Functions

```solidity
hasRole(bytes32 role, address account) → bool
grantRole(bytes32 role, address account)    // only callable by adminRole holder
revokeRole(bytes32 role, address account)   // only callable by adminRole holder
renounceRole(bytes32 role, address account) // self-revoke only (account == msg.sender)
getRoleAdmin(bytes32 role) → bytes32        // returns the adminRole for a role
```

### `onlyRole` Modifier

```solidity
modifier onlyRole(bytes32 role) {
    _checkRole(role);
    _;
}

function _checkRole(bytes32 role) internal view {
    if (!hasRole(role, msg.sender)) {
        revert AccessControlUnauthorizedAccount(msg.sender, role);
    }
}
```

Two SLOADs: one for the `RoleData` struct and one for the `hasRole` mapping lookup.

### AccessControlEnumerable

Extends `AccessControl` with on-chain enumeration. Adds a `EnumerableSet.AddressSet` per role so you can iterate all holders:

```solidity
getRoleMemberCount(bytes32 role) → uint256
getRoleMember(bytes32 role, uint256 index) → address
```

**Gas cost:** Every `grantRole`/`revokeRole` also updates the set (extra SSTOREs). Only use when you need on-chain enumeration — most protocols don't.

---

## 4. AccessControlDefaultAdminRules

### Why It Exists

`DEFAULT_ADMIN_ROLE` is the master key — whoever holds it can grant themselves any role. A stolen admin key or careless `grantRole(DEFAULT_ADMIN_ROLE, attacker)` is catastrophic. This contract hardens that one role with:

1. **2-step transfer** (like `Ownable2Step`)
2. **Time delay** before the transfer takes effect
3. **Cancellation window** — admin can cancel a pending transfer
4. **Enforced single admin** — only one address can hold `DEFAULT_ADMIN_ROLE` at a time

```solidity
contract MyContract is AccessControlDefaultAdminRules {
    constructor()
        AccessControlDefaultAdminRules(
            3 days,       // delay before new admin can accept
            msg.sender    // initial admin
        )
    {}
}
```

### Internal Storage (additional over AccessControl)

```solidity
address private _currentDefaultAdmin;
uint48 private _currentDelay;

address private _pendingDefaultAdmin;
uint48 private _pendingDefaultAdminSchedule; // timestamp when accept becomes valid

address private _pendingDelay;               // proposed new delay
uint48 private _pendingDelaySchedule;
```

### Admin Transfer Flow

```
Day 0:  admin calls beginDefaultAdminTransfer(newAdmin)
          → stores newAdmin in _pendingDefaultAdmin
          → sets schedule = block.timestamp + delay
          → emits DefaultAdminTransferScheduled

Day 3+: newAdmin calls acceptDefaultAdminTransfer()
          → verifies block.timestamp >= schedule
          → removes DEFAULT_ADMIN_ROLE from old admin
          → grants DEFAULT_ADMIN_ROLE to newAdmin
          → clears pending state
```

Any time before acceptance, the current admin can call `cancelDefaultAdminTransfer()` to abort.

### Changing the Delay Itself

Changing the delay also has a guard to prevent an admin from instantly dropping the delay to 0:

```solidity
beginDefaultAdminDelayChange(uint48 newDelay)  // propose new delay
acceptDefaultAdminDelayChange()                 // apply after current delay has passed
cancelDefaultAdminDelayChange()                // cancel proposal
```

### DefaultAdminRules Functions

```solidity
defaultAdmin() → address                         // current DEFAULT_ADMIN_ROLE holder
pendingDefaultAdmin() → (address, uint48)        // (pending admin, schedule timestamp)
defaultAdminDelay() → uint48                     // current delay in seconds
beginDefaultAdminTransfer(address newAdmin)      // step 1
acceptDefaultAdminTransfer()                     // step 2 (after delay)
cancelDefaultAdminTransfer()                     // abort pending transfer
```

> **Important:** `grantRole(DEFAULT_ADMIN_ROLE, anyone)` and `revokeRole(DEFAULT_ADMIN_ROLE, anyone)` are blocked — they revert. You must go through the `beginDefaultAdminTransfer` flow.

---

## 5. Upgradeable Counterparts

### What Changes in Upgradeable Variants

All upgradeable OZ contracts follow the same pattern:

1. **No constructor** — replace with an `__X_init()` initializer function
2. **`__gap` storage variable** — reserved slots for future storage additions without breaking inherited layout
3. **Must call parent initializers** — via `__X_init_unchained()` or `__X_init()`

| Non-Upgradeable | Upgradeable |
| --- | --- |
| `Ownable` | `OwnableUpgradeable` |
| `Ownable2Step` | `Ownable2StepUpgradeable` |
| `AccessControl` | `AccessControlUpgradeable` |
| `AccessControlDefaultAdminRules` | `AccessControlDefaultAdminRulesUpgradeable` |

### OwnableUpgradeable

```solidity
contract MyProxy is Initializable, OwnableUpgradeable {
    function initialize(address initialOwner) public initializer {
        __Ownable_init(initialOwner);  // sets _owner, replaces constructor arg
    }
}
```

Storage:
```solidity
// OwnableUpgradeable.sol
address private _owner;
uint256[49] private __gap;  // 49 reserved slots
```

`__gap` ensures that if OZ adds a new storage variable to `OwnableUpgradeable` in a future version, it goes into the gap without shifting storage of derived contracts.

### Ownable2StepUpgradeable

```solidity
function initialize(address initialOwner) public initializer {
    __Ownable2Step_init();             // calls __Ownable_init internally
    // Note: __Ownable_init(initialOwner) must be called somewhere in the chain
}
```

Storage:
```solidity
address private _owner;          // from OwnableUpgradeable
uint256[49] private __gap;       // from OwnableUpgradeable

address private _pendingOwner;   // from Ownable2StepUpgradeable
uint256[49] private __gap;       // from Ownable2StepUpgradeable
```

### AccessControlUpgradeable

```solidity
contract MyProxy is Initializable, AccessControlUpgradeable {
    bytes32 public constant MINTER_ROLE = keccak256("MINTER_ROLE");

    function initialize() public initializer {
        __AccessControl_init();
        _grantRole(DEFAULT_ADMIN_ROLE, msg.sender);
    }
}
```

Storage:
```solidity
mapping(bytes32 role => AccessControl.RoleData) private _roles;
uint256[49] private __gap;
```

### AccessControlDefaultAdminRulesUpgradeable

```solidity
contract MyProxy is Initializable, AccessControlDefaultAdminRulesUpgradeable {
    function initialize(address initialAdmin) public initializer {
        __AccessControlDefaultAdminRules_init(3 days, initialAdmin);
    }
}
```

All the same constraints apply — `DEFAULT_ADMIN_ROLE` grant/revoke is blocked, transfer requires 2-step + delay.

### Critical: Initializer Must Be Called Once

```solidity
modifier initializer() {
    // reverts if already initialized
    // ensures init functions can't be called again after deployment
}
```

A proxy's `initialize()` function **must be called in the same transaction as deployment**, or anyone can call it first and become the admin. This is a common exploit vector.

```solidity
// Safe deployment (Foundry example)
MyProxy proxy = new MyProxy();
proxy.initialize(adminAddress);  // must be in same tx or atomically via factory
```

---

## Security Considerations

### Ownable / Ownable2Step

- **Compromised owner key** — single point of failure; consider a multisig as owner
- **`renounceOwnership` is not 2-step** — a griefing owner (or compromised key) can permanently brick the contract in one tx even with `Ownable2Step`
- **Ownership transfer to a contract** — ensure the receiving contract has `acceptOwnership()` logic, otherwise ownership is permanently stuck (more relevant for `Ownable2Step`)

### AccessControl

- **Granting `DEFAULT_ADMIN_ROLE` to EOA** — should usually be a multisig or timelock in production
- **Role admin misconfiguration** — forgetting to set `_setRoleAdmin` means `DEFAULT_ADMIN_ROLE` controls all roles (sometimes intended, sometimes not)
- **`renounceRole` vs `revokeRole`** — `renounceRole` requires `msg.sender == account` (self-only). `revokeRole` is called by the role's admin. Don't confuse them.
- **Losing `DEFAULT_ADMIN_ROLE`** — if no address holds it, no one can grant new roles. Permanently frozen permissions.
- **Multiple admin holders** — `AccessControl` allows multiple `DEFAULT_ADMIN_ROLE` holders. Each is equally powerful.

### AccessControlDefaultAdminRules

- **Time delay is global** — the delay applies to every admin transfer, even emergency ones. Set a delay you can live with in a crisis.
- **Delay change is also delayed** — if you set a 30-day delay and need to lower it urgently, the new lower delay itself takes 30 days to take effect.
- **Still need a multisig as admin** — the 2-step + delay prevents accidental transfers but doesn't protect against a fully compromised key. Combine with a hardware-wallet multisig.

### Upgradeable Variants (Additional)

- **Uninitialized implementation contract** — the implementation contract itself (not the proxy) has no owner set. An attacker can call `initialize()` on the implementation directly and `selfdestruct` it via `delegatecall` (in older Solidity). Always initialize or disable initializers on the implementation:

  ```solidity
  constructor() {
      _disableInitializers();
  }
  ```

- **Storage collision** — if you change the order of inherited contracts between upgrades, storage slots shift. Always verify layout with tools like `slither --detect storage-collision` or OpenZeppelin's Upgrades plugin.
- **`__gap` size** — if a base contract exhausts its `__gap` by adding state variables in an upgrade, derived contracts' storage shifts. Track how many slots each base contract's `__gap` has.
- **Re-initialization** — `initializer` modifier blocks re-calling `initialize()`. But `reinitializer(version)` allows a controlled re-init for upgrade migrations. Be careful not to accidentally expose one.

---

## Roles vs Whitelists — When to Use What

**AccessControl roles** are for protocol operators (small set, high privilege):
```
PAUSER_ROLE         → ops team multisig (2-3 addresses)
ORACLE_ROLE         → oracle adapter contract
UPGRADER_ROLE       → governance timelock
```

**Custom mappings** are for user-level access (larger set, single gate):
```solidity
mapping(address => bool) public whitelistedCreators;
```

**Rule of thumb:**
```
Small set, high privilege, manages the protocol  → AccessControl role
Larger set, single purpose, uses the protocol    → mapping whitelist
Very large set, gas-sensitive                    → Merkle proof
```

---

## Quick Reference

```solidity
// ── Ownable ──────────────────────────────────────────────────────────
owner()
transferOwnership(newOwner)          // direct, immediate
renounceOwnership()                  // permanent, immediate

// ── Ownable2Step ─────────────────────────────────────────────────────
owner()
pendingOwner()
transferOwnership(newOwner)          // sets pendingOwner, does NOT change owner yet
acceptOwnership()                    // called by pendingOwner to finalize
renounceOwnership()                  // still direct (not 2-step!)

// ── AccessControl ─────────────────────────────────────────────────────
DEFAULT_ADMIN_ROLE                   // bytes32(0)
hasRole(role, account) → bool
grantRole(role, account)             // only adminRole holder of that role
revokeRole(role, account)            // only adminRole holder of that role
renounceRole(role, account)          // self-revoke (account == msg.sender)
getRoleAdmin(role) → bytes32

// ── AccessControlDefaultAdminRules ────────────────────────────────────
defaultAdmin() → address
pendingDefaultAdmin() → (address, uint48 schedule)
defaultAdminDelay() → uint48

beginDefaultAdminTransfer(newAdmin)
acceptDefaultAdminTransfer()         // only after delay passes
cancelDefaultAdminTransfer()

beginDefaultAdminDelayChange(newDelay)
acceptDefaultAdminDelayChange()
cancelDefaultAdminDelayChange()

// ── Upgradeable init replacements ────────────────────────────────────
__Ownable_init(initialOwner)
__Ownable2Step_init()
__AccessControl_init()
__AccessControlDefaultAdminRules_init(delay, initialAdmin)
```
