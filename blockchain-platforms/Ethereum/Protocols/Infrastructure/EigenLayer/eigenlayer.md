# EigenLayer Protocol

## Table of Contents

- [Overview](#overview)
  - [What is EigenLayer?](#what-is-eigenlayer)
  - [Core Value Proposition](#core-value-proposition)
  - [Architecture Overview](#architecture-overview)
  - [How It Works](#how-it-works)
- [Restaking](#restaking)
  - [What is Restaking?](#what-is-restaking)
  - [Native Restaking](#native-restaking)
  - [Liquid Restaking (LST)](#liquid-restaking-lst)
  - [Economic Model](#economic-model)
  - [Restaking Security Considerations](#restaking-security-considerations)
- [AVS (Actively Validated Services)](#avs-actively-validated-services)
  - [What are AVSs?](#what-are-avss)
  - [AVS Architecture](#avs-architecture)
  - [AVS Registration Flow](#avs-registration-flow)
  - [Operator-AVS Relationship](#operator-avs-relationship)
  - [Examples of AVS Use Cases](#examples-of-avs-use-cases)
- [Operators](#operators)
  - [Operator Role](#operator-role)
  - [Operator Responsibilities](#operator-responsibilities)
  - [Delegation System](#delegation-system)
  - [Operator Economics](#operator-economics)
  - [Operator Security](#operator-security)
- [Slashing](#slashing)
  - [Slashing Overview](#slashing-overview)
  - [Slashing Conditions](#slashing-conditions)
  - [Slashing Process](#slashing-process)
  - [Veto Committee](#veto-committee)
  - [Slashing Security Considerations](#slashing-security-considerations)
- [Smart Contract Architecture](#smart-contract-architecture)
  - [Key Contracts](#key-contracts)
  - [Contract Interfaces](#contract-interfaces)
  - [Contract Interactions](#contract-interactions)
- [Resources](#resources)

---

## Overview

### What is EigenLayer?

EigenLayer is a **restaking protocol** built on Ethereum that enables crypto-economic security sharing. It allows Ethereum validators and stakers to reuse their staked ETH to secure additional decentralized services beyond just the Ethereum blockchain itself.

**Key Concepts:**

- **Restaking**: The ability to stake the same ETH across multiple protocols
- **Shared Security**: New services can leverage Ethereum's existing validator set and economic security
- **Opt-in Model**: Validators choose which additional services (AVSs) to validate

### Core Value Proposition

EigenLayer solves several key problems in the blockchain ecosystem:

1. **Capital Efficiency**
   - Stakers can earn additional yield on their staked ETH
   - No need to unstake or maintain separate validator sets

2. **Lower Barrier for New Services**
   - New decentralized services (data availability layers, oracles, bridges) don't need to bootstrap their own validator set from scratch
   - Can instantly access Ethereum's $XX billion in staked capital

3. **Programmable Trust**
   - Services define their own slashing conditions
   - Flexible security model tailored to each service's needs

4. **Composable Security**
   - Build on top of Ethereum's battle-tested consensus
   - Inherit Ethereum's decentralization and liveness guarantees

### Architecture Overview

EigenLayer consists of three main participants:

```
┌─────────────────────────────────────────────────────────────┐
│                      EigenLayer Ecosystem                   │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────┐         ┌──────────┐         ┌──────────┐      │
│  │ Stakers │────────>│Operators │<───────>│   AVSs   │      │
│  └─────────┘         └──────────┘         └──────────┘      │
│       │                    │                     │          │
│       │                    │                     │          │
│       ▼                    ▼                     ▼          │
│  ┌─────────────────────────────────────────────────────┐    │
│  │         EigenLayer Smart Contracts                  │    │
│  │  ┌────────────┐  ┌────────────┐  ┌──────────────┐   │    │
│  │  │  Strategy  │  │ Delegation │  │      AVS     │   │    │
│  │  │  Manager   │  │  Manager   │  │   Directory  │   │    │
│  │  └────────────┘  └────────────┘  └──────────────┘   │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**Three Key Participants:**

1. **Stakers**: ETH holders who deposit into EigenLayer to earn additional yield
2. **Operators**: Entities that run validation software for AVSs and accept delegations from stakers
3. **AVSs (Actively Validated Services)**: Decentralized services that need validation (data availability, oracles, sequencers, etc.)

### How It Works

**High-Level Flow:**

1. **Stakers deposit** ETH or liquid staking tokens (LSTs) into EigenLayer contracts
2. **Stakers delegate** their stake to Operators they trust
3. **Operators opt into** specific AVSs they want to validate
4. **AVSs define** their slashing conditions and validation requirements
5. **Security propagates** from Ethereum's base layer through restaking to the AVS

**Example Flow:**

```
1. Alice deposits 32 ETH into EigenLayer
   └─> Receives shares representing her restaked position

2. Alice delegates to Operator Bob
   └─> Bob now has 32 ETH of delegated stake

3. Bob opts into AVS "DataAvailability Layer"
   └─> Runs the AVS validation software
   └─> Can be slashed if misbehaves

4. AVS pays Bob for validation services
   └─> Bob takes commission and passes rewards to Alice

5. If Bob misbehaves, both Bob and Alice can be slashed
   └─> Economic security enforces correct behavior
```

---

## Restaking

### What is Restaking?

**Restaking** allows Ethereum stakers to reuse their staked ETH to provide economic security to additional protocols and services beyond Ethereum itself.

**Traditional Staking:**

- Stake ETH → Secure Ethereum blockchain → Earn ETH rewards

**Restaking:**

- Stake ETH → Secure Ethereum blockchain AND additional services → Earn ETH rewards + AVS rewards

**Two Types:**

1. **Native Restaking**: Using ETH from Ethereum validators directly
2. **Liquid Restaking**: Using liquid staking tokens (LSTs) like stETH, rETH, cbETH

### Native Restaking

Native restaking allows Ethereum validators to redirect their withdrawal credentials to EigenLayer's smart contracts called **EigenPods**.

**How It Works:**

```solidity
// Validator creates an EigenPod
interface IEigenPodManager {
    /// @notice Creates an EigenPod for the sender
    function createPod() external returns (address);

    /// @notice Stakes on behalf of pod owner
    function stake(bytes calldata pubkey, bytes calldata signature, bytes32 depositDataRoot) external payable;
}
```

**Process:**

1. **Create EigenPod**

   ```
   Validator → EigenPodManager.createPod()
   └─> Deploys EigenPod contract for validator
   ```

2. **Set Withdrawal Credentials**

   ```
   ETH Validator → Points withdrawal credentials to EigenPod address
   ```

3. **Prove Validator Balance**

   ```
   Operator → Submits Beacon Chain proof to EigenPod
   └─> Verifies validator balance and activates restaking
   ```

**EigenPod Key Functions:**

```solidity
interface IEigenPod {
    /// @notice Verify withdrawal credentials of validator(s)
    function verifyWithdrawalCredentials(
        uint64 oracleTimestamp,
        BeaconChainProofs.StateRootProof calldata stateRootProof,
        uint40[] calldata validatorIndices,
        bytes[] calldata validatorFieldsProofs,
        bytes32[][] calldata validatorFields
    ) external;

    /// @notice Verify and process withdrawals
    function verifyAndProcessWithdrawals(
        uint64 oracleTimestamp,
        BeaconChainProofs.StateRootProof calldata stateRootProof,
        BeaconChainProofs.WithdrawalProof[] calldata withdrawalProofs,
        bytes[] calldata validatorFieldsProofs,
        bytes32[][] calldata validatorFields,
        bytes32[][] calldata withdrawalFields
    ) external;
}
```

#### What is an EigenPod?

An **EigenPod** is a smart contract that serves as the withdrawal address for Ethereum validators participating in native restaking. It acts as a bridge between the Ethereum Beacon Chain and EigenLayer, allowing validators to restake their ETH without giving up custody of their validator keys.

**Key Characteristics:**

- **One EigenPod per address**: Each Ethereum address can create one EigenPod
- **Withdrawal credential target**: The EigenPod address becomes the validator's withdrawal credential
- **Proof-based accounting**: Uses Beacon Chain state proofs to track validator balances
- **Non-custodial**: Validator retains full control of their validator keys

**EigenPod Architecture:**

```
Ethereum Beacon Chain                 EigenLayer
────────────────────                  ──────────

┌──────────────┐
│   Validator  │
│   (32 ETH)   │
└──────┬───────┘
       │
       │ Withdrawal Credentials
       │ point to ↓
       │
       ▼
┌──────────────────┐                 ┌────────────────┐
│    EigenPod      │ ───────────────>│  EigenLayer    │
│   (Smart         │   Shares         │  Delegation    │
│    Contract)     │   Accounting     │  Manager       │
└──────────────────┘                 └────────────────┘
       │
       │ Proofs submitted to verify
       │ validator state
       ▼
┌──────────────────┐
│  Beacon Chain    │
│  Oracle          │
│  (State Proofs)  │
└──────────────────┘
```

#### EigenPod Lifecycle

**1. Creation**

Anyone can create an EigenPod for their address:

```solidity
interface IEigenPodManager {
    /// @notice Creates an EigenPod for msg.sender
    /// @return The address of the created EigenPod
    function createPod() external returns (address);

    /// @notice Returns the EigenPod of a given address
    function ownerToPod(address owner) external view returns (address);

    /// @notice Returns the pod owner of a given EigenPod
    function podOwnerShares(address podOwner) external view returns (int256);
}
```

**Process:**

```
User calls EigenPodManager.createPod()
    └─> Deploys minimal proxy contract (ERC-1967)
        └─> Points to EigenPod implementation
            └─> Returns EigenPod address (e.g., 0xABC...123)
```

**2. Validator Setup**

After creating an EigenPod, set it as the withdrawal credential:

```bash
# When creating a new validator
eth2-deposit-cli --withdrawal_address <EIGENPOD_ADDRESS>

# For existing validators (requires 0x01 credentials upgrade)
# Must wait for Shanghai/Capella upgrade feature
```

**Withdrawal Credential Format:**

```
Old (BLS): 0x00 + 31 bytes (BLS pubkey hash)
New (ETH): 0x01 + 11 bytes (padding) + 20 bytes (EigenPod address)
```

**3. Proof Submission**

After validators are active, submit proofs to activate restaking:

```solidity
interface IEigenPod {
    /// @notice Verify that validator(s) have their withdrawal credentials pointed to this EigenPod
    function verifyWithdrawalCredentials(
        uint64 oracleTimestamp,
        BeaconChainProofs.StateRootProof calldata stateRootProof,
        uint40[] calldata validatorIndices,
        bytes[] calldata validatorFieldsProofs,
        bytes32[][] calldata validatorFields
    ) external;
}
```

**What This Proves:**

- Validator exists on Beacon Chain
- Validator's withdrawal credentials point to this EigenPod
- Validator's balance (used for restaking shares calculation)

**4. Balance Updates**

Validators' balances change over time (rewards, slashing). EigenPods track this:

```solidity
interface IEigenPod {
    /// @notice Verify validator balance updates
    function verifyBalanceUpdates(
        uint64 oracleTimestamp,
        uint40[] calldata validatorIndices,
        BeaconChainProofs.StateRootProof calldata stateRootProof,
        bytes[] calldata validatorFieldsProofs,
        bytes32[][] calldata validatorFields
    ) external;
}
```

**Balance Accounting:**

```
Initial Proof: Validator has 32 ETH → 32 shares minted
After 1 year:  Validator has 33.5 ETH → Additional 1.5 shares minted
If slashed:    Validator has 31 ETH → 1 share burned (from 33.5)
```

**5. Withdrawals**

When validators exit or perform partial withdrawals:

```solidity
interface IEigenPod {
    /// @notice Verify and process Beacon Chain withdrawals
    function verifyAndProcessWithdrawals(
        uint64 oracleTimestamp,
        BeaconChainProofs.StateRootProof calldata stateRootProof,
        BeaconChainProofs.WithdrawalProof[] calldata withdrawalProofs,
        bytes[] calldata validatorFieldsProofs,
        bytes32[][] calldata validatorFields,
        bytes32[][] calldata withdrawalFields
    ) external;

    /// @notice Withdraw ETH from the pod (after restaking deactivated)
    function withdrawBeforeRestaking() external;

    /// @notice Activate restaking (irreversible)
    function activateRestaking() external;
}
```

**Withdrawal Types:**

```
Partial Withdrawal (>32 ETH balance):
- Automatically sent to EigenPod
- Can be withdrawn by pod owner
- Used to pay out staking rewards

Full Withdrawal (validator exit):
- All validator ETH sent to EigenPod
- Must queue withdrawal through EigenLayer
- Subject to withdrawal delay period
```

#### Beacon Chain Proofs

EigenPods rely on **cryptographic proofs** of Beacon Chain state. This enables trustless verification without oracles.

**Proof Components:**

```solidity
library BeaconChainProofs {
    struct StateRootProof {
        bytes32 beaconStateRoot;
        bytes proof; // Merkle proof
    }

    struct ValidatorProof {
        uint40 validatorIndex;
        bytes32 validatorRoot;
        bytes proof; // Merkle proof from state root to validator
    }

    struct WithdrawalProof {
        bytes32 withdrawalRoot;
        bytes proof; // Merkle proof
    }
}
```

**How Proofs Work:**

```
Beacon Chain State Tree:
└─ State Root (published to execution layer)
   ├─ Validators
   │  └─ Validator[index]
   │     ├─ Public Key
   │     ├─ Withdrawal Credentials ← Proves this points to EigenPod
   │     ├─ Balance ← Proves validator's current balance
   │     └─ Status (active, exited, slashed)
   │
   └─ Withdrawals
      └─ Withdrawal[index]
         ├─ Validator Index
         ├─ Amount ← Proves withdrawal amount
         └─ Address ← Proves destination (must be EigenPod)

EigenPod verifies Merkle proofs against on-chain Beacon Chain state root
```

**Security Model:**

- State roots posted to Execution Layer via Beacon Block Root contract
- Merkle proofs are cryptographically verified on-chain
- No trust in external oracles
- Proofs can be submitted by anyone (permissionless)

#### EigenPod State Machine

EigenPods track validator state through a state machine:

```solidity
enum VALIDATOR_STATUS {
    INACTIVE,           // Not yet proven to EigenPod
    ACTIVE,            // Proven and actively restaking
    WITHDRAWN          // Validator has fully withdrawn
}

struct ValidatorInfo {
    uint64 validatorIndex;
    uint64 restakedBalanceGwei;
    uint64 mostRecentBalanceUpdateTimestamp;
    VALIDATOR_STATUS status;
}
```

**State Transitions:**

```
INACTIVE
   │
   │ verifyWithdrawalCredentials()
   ▼
ACTIVE ─────────────────────────────┐
   │                                │
   │ verifyBalanceUpdates()         │
   │ (balance increases/decreases)  │
   │                                │
   │                                │ verifyAndProcessWithdrawals()
   │                                │ (full withdrawal)
   ▼                                ▼
ACTIVE (updated balance)         WITHDRAWN
```

#### EigenPod Economics

**Share Calculation:**

Shares represent the validator's restaked balance:

```
Initial Verification:
Validator Balance: 32 ETH
Shares Minted: 32 shares (1:1 ratio initially)

After Rewards:
Validator Balance: 33.5 ETH
Additional Shares: 1.5 shares minted
Total Shares: 33.5 shares

After Slashing:
Validator Balance: 31 ETH (slashed 2.5 ETH)
Shares Burned: 2.5 shares
Total Shares: 31 shares
```

**Shares are used for:**

- Delegation weight to operators
- Reward distribution
- Slashing calculations
- Withdrawal amounts

**Pod Owner Shares Accounting:**

```solidity
interface IEigenPodManager {
    /// @notice Returns the shares for a pod owner (can be negative!)
    function podOwnerShares(address podOwner) external view returns (int256);
}
```

**Note:** Shares can be **negative** if validator is slashed below certain thresholds!

#### EigenPod Security Considerations

**1. Withdrawal Credential Irreversibility**

Once you set your validator's withdrawal credentials to an EigenPod, **this cannot be changed** without fully exiting the validator.

```
Risk:
- If EigenPod contract has bugs, funds could be locked
- If EigenLayer is compromised, restaked ETH at risk

Mitigation:
- Extensive audits of EigenPod contracts
- Formal verification of core logic
- Gradual rollout and testing
- Insurance protocols
```

**2. Proof Submission Timing**

Stale proofs could lead to incorrect accounting:

```solidity
// EigenPods enforce proof freshness
uint256 constant VERIFY_BALANCE_UPDATE_WINDOW_SECONDS = 4.5 hours;

function verifyBalanceUpdates(...) external {
    require(
        oracleTimestamp + VERIFY_BALANCE_UPDATE_WINDOW_SECONDS >= block.timestamp,
        "Stale proof"
    );
    // ...
}
```

**3. Beacon Chain Reorgs**

Short reorgs on Beacon Chain could invalidate proofs:

```
Mitigation:
- Wait for sufficient finality before submitting proofs
- Proofs reference finalized state roots
- Reorg depth protection in contracts
```

**4. Slashing on Both Layers**

Validators face slashing on both Ethereum and EigenLayer:

```
Compound Slashing Risk:
Ethereum Slashing (0.5-1 ETH typical)
     +
EigenLayer Slashing (AVS-dependent)
     =
Potentially significant total loss
```

**Benefits:**

- Direct control of validator keys
- Maximum capital efficiency
- No intermediary token
- Non-custodial (retain validator key control)
- Trustless balance verification via proofs

**Considerations:**

- More complex setup than LST restaking
- Requires running an Ethereum validator
- Must manage validator keys carefully
- Withdrawal credentials change is irreversible
- Requires understanding of Beacon Chain proofs
- Must submit proofs to maintain accurate accounting

### Liquid Restaking (LST)

Liquid Restaking allows users to deposit liquid staking tokens (LSTs) into EigenLayer without running a validator.

**Supported LSTs:**

- stETH (Lido)
- rETH (Rocket Pool)
- cbETH (Coinbase)
- And others...

**How It Works:**

```solidity
interface IStrategyManager {
    /// @notice Deposits tokens into a specified strategy
    /// @param strategy The strategy to deposit into
    /// @param token The token to deposit
    /// @param amount The amount to deposit
    function depositIntoStrategy(
        IStrategy strategy,
        IERC20 token,
        uint256 amount
    ) external returns (uint256 shares);

    /// @notice Queue withdrawals from strategies
    function queueWithdrawals(
        QueuedWithdrawalParams[] calldata queuedWithdrawalParams
    ) external returns (bytes32[] memory);

    /// @notice Complete queued withdrawals
    function completeQueuedWithdrawal(
        Withdrawal calldata withdrawal,
        IERC20[] calldata tokens,
        uint256 middlewareTimesIndex,
        bool receiveAsTokens
    ) external;
}
```

**Process:**

1. **Deposit LST**

   ```
   User → StrategyManager.depositIntoStrategy(stETH_strategy, stETH, amount)
   └─> Receives EigenLayer shares
   ```

2. **Shares Accounting**

   ```solidity
   interface IStrategy {
       /// @notice Calculates shares for a given amount
       function underlyingToShares(uint256 amountUnderlying) external view returns (uint256);

       /// @notice Calculates tokens for a given share amount
       function sharesToUnderlying(uint256 amountShares) external view returns (uint256);
   }
   ```

3. **Withdrawal Flow**

   ```
   User → Queue withdrawal (enters unbonding period)
   └─> Wait for withdrawal delay
   └─> Complete withdrawal (receive tokens back)
   ```

**Benefits:**

- Easy onboarding (just deposit LSTs)
- No need to run validator infrastructure
- Composable with DeFi

**Considerations:**

- Depends on underlying LST protocol
- Additional smart contract risk
- May have withdrawal delays

### Economic Model

**Yield Sources:**

1. **Base Ethereum Staking Rewards** (~3-4% APR)
   - Consensus layer rewards
   - Execution layer rewards (MEV)

2. **AVS Rewards** (Variable)
   - Payment from AVSs for validation services
   - Can be paid in ETH, AVS tokens, or other assets

**Risk/Reward Tradeoff:**

```
Higher Yield = Higher Risk

No Restaking:     3-4% APR  │ Only Ethereum slashing risk
                            │
Restaking (1 AVS): 5-7% APR │ Ethereum + 1 AVS slashing risk
                            │
Restaking (5 AVS): 8-12% APR│ Ethereum + 5 AVS slashing risks
```

**Slashing Multiplier Effect:**

When restaking across multiple AVSs, slashing risks compound:

```
Scenario: Operator validates 3 AVSs with 100 ETH stake

AVS A slashing condition triggered: -10 ETH
AVS B slashing condition triggered: -15 ETH
AVS C slashing condition triggered: -20 ETH

Maximum possible loss: -45 ETH (45% of stake)
```

### Restaking Security Considerations

**1. Increased Slashing Risk**

Every AVS you opt into adds additional slashing conditions:

```solidity
// Each AVS can define custom slashing logic
interface IAVSSlasher {
    function slashOperator(
        address operator,
        uint256 amount,
        string calldata reason
    ) external;
}
```

**Mitigation:**

- Carefully vet AVSs before opting in
- Understand each AVS's slashing conditions
- Limit number of concurrent AVSs
- Use stake allocation limits

**2. Operator Selection Risk**

Stakers delegate to operators who make validation decisions:

```
Bad Operator Scenario:
- Opts into risky AVSs without staker knowledge
- Runs low-quality infrastructure (high downtime)
- Gets slashed → Delegated stakers lose funds
```

**Mitigation:**

- Research operator reputation and track record
- Check operator's AVS opt-ins
- Monitor operator performance
- Diversify across multiple operators

**3. Correlation Risk**

Multiple AVSs may have correlated failure modes:

```
Example: Multiple data availability AVSs
- All depend on similar infrastructure
- Network partition affects all simultaneously
- Correlated slashing across all AVSs
```

**Mitigation:**

- Diversify across different AVS types
- Understand AVS dependencies
- Avoid overcorrelation with single failure points

**4. Smart Contract Risk**

Additional smart contract surface area:

- EigenLayer core contracts
- AVS contracts
- Strategy contracts for each LST
- Integration bugs

**Mitigation:**

- Wait for battle-tested contracts
- Review audit reports
- Start with small amounts
- Use insurance protocols if available

---

## AVS (Actively Validated Services)

### What are AVSs?

**Actively Validated Services (AVSs)** are decentralized services that require active validation by operators. They leverage EigenLayer's restaking infrastructure to bootstrap economic security without needing to create their own validator set.

**Examples of AVSs:**

- Data availability layers
- Oracle networks
- Cross-chain bridges
- Decentralized sequencers
- Fast finality layers
- MEV mitigation services
- Threshold cryptography networks

**Why AVSs Need Validation:**

Traditional decentralized services face the "cold start" problem:

```
New Service Launch:
1. Need validators for security
2. Validators need incentives to join
3. Incentives require protocol value/revenue
4. Value requires users and adoption
5. Users need security guarantees ← (Circular dependency)
```

EigenLayer breaks this cycle by providing instant access to Ethereum's validator set and security.

### AVS Architecture

An AVS consists of three components:

```
┌────────────────────────────────────────────┐
│              AVS Architecture              │
├────────────────────────────────────────────┤
│                                            │
│  ┌──────────────────────────────────────┐  │
│  │     1. On-Chain Contracts            │  │
│  │  - Service registry                  │  │
│  │  - Slashing logic                    │  │
│  │  - Payment distribution              │  │
│  └──────────────────────────────────────┘  │
│                    │                       │
│                    ▼                       │
│  ┌──────────────────────────────────────┐  │
│  │     2. Off-Chain Validation Logic    │  │
│  │  - Validation software run by        |  |
|  |    operators                         │  │
│  │  - Task assignment & execution       │  │
│  │  - Proof generation                  │  │
│  └──────────────────────────────────────┘  │
│                    │                       │
│                    ▼                       │
│  ┌──────────────────────────────────────┐  │
│  │     3. Task Distribution System      │  │
│  │  - Assigns work to operators         │  │
│  │  - Verifies completion               │  │
│  │  - Triggers slashing if needed       │  │
│  └──────────────────────────────────────┘  │
│                                            │
└────────────────────────────────────────────┘
```

**Example: Data Availability AVS**

```solidity
interface IDataAvailabilityAVS {
    struct Blob {
        bytes32 commitment;
        bytes data;
        bytes proof;
    }

    /// @notice Submit blob for validation
    function submitBlob(Blob calldata blob) external;

    /// @notice Operators attest to data availability
    function attestAvailability(bytes32 blobHash, bytes calldata signature) external;

    /// @notice Challenge unavailable data
    function challengeAvailability(bytes32 blobHash, bytes calldata proof) external;
}
```

### AVS Registration Flow

**Step 1: AVS Registers with EigenLayer**

```solidity
interface IAVSDirectory {
    struct AVSMetadata {
        string name;
        string description;
        string logo;
        string website;
    }

    /// @notice Register new AVS
    function registerAVS(
        address avsOperator,
        string calldata metadataURI
    ) external;

    /// @notice Update AVS metadata
    function updateAVSMetadataURI(string calldata metadataURI) external;
}
```

**Step 2: Define Slashing Conditions**

```solidity
interface IAVSServiceManager {
    /// @notice Called when slashing should occur
    function freezeOperator(address operator) external;

    /// @notice Defines what constitutes slashable behavior
    struct SlashingCondition {
        bytes32 conditionHash;
        uint256 slashAmount;
        string description;
    }

    function getSlashingConditions() external view returns (SlashingCondition[] memory);
}
```

**Step 3: Operators Opt In**

```solidity
interface IAVSDirectory {
    /// @notice Operator opts into AVS
    function registerOperatorToAVS(
        address operator,
        address avs,
        ISignatureUtils.SignatureWithSaltAndExpiry memory operatorSignature
    ) external;

    /// @notice Operator opts out of AVS
    function deregisterOperatorFromAVS(
        address operator,
        address avs
    ) external;
}
```

### Operator-AVS Relationship

**Opt-in Mechanism:**

Operators explicitly choose which AVSs to validate:

```
Operator Decision Process:
1. Review AVS requirements (hardware, bandwidth, etc.)
2. Assess slashing conditions and risk
3. Evaluate payment/rewards
4. Opt in via AVSDirectory.registerOperatorToAVS()
5. Run AVS validation software
6. Continuously monitor and validate
```

**Service Level Agreements:**

AVSs and operators have implicit/explicit agreements:

```solidity
interface IAVSServiceManager {
    struct ServiceAgreement {
        uint256 minimumStake;        // Min stake required
        uint256 rewardRate;           // Payment per task/time
        uint256 slashingMultiplier;   // Slashing severity
        uint256 minimumLatency;       // Max response time
        uint256 minimumAvailability;  // Uptime requirement (basis points)
    }

    function getServiceAgreement() external view returns (ServiceAgreement memory);
}
```

**Payment Flows:**

```
AVS Revenue Sources:
├─> Protocol fees from users
├─> Token emissions
└─> Third-party subsidies

Payment Distribution:
AVS Revenue → Service Manager Contract
                    │
                    ├──> Operators (validation rewards)
                    │       │
                    │       └──> Delegated Stakers (minus operator commission)
                    │
                    └──> Protocol treasury/development
```

### Examples of AVS Use Cases

#### 1. Data Availability Layer

**Problem:** Rollups need to publish data but Ethereum L1 is expensive

**EigenLayer Solution:**

```
EigenDA (Data Availability AVS):
- Operators store and attest to blob availability
- Cheaper than Ethereum calldata
- Slashing if data becomes unavailable
- Dispersal and retrieval protocols
```

**Smart Contract:**

```solidity
interface IEigenDA {
    function storeBlob(bytes calldata data) external returns (bytes32 commitment);

    function verifyBlobInclusion(
        bytes32 commitment,
        bytes calldata proof
    ) external view returns (bool);
}
```

#### 2. Decentralized Sequencer

**Problem:** Rollup sequencers are currently centralized

**EigenLayer Solution:**

```
Sequencer AVS:
- Multiple operators share sequencing duties
- Fast pre-confirmations before L1 finality
- Slashing for invalid sequencing
- Byzantine fault tolerant consensus
```

#### 3. Oracle Network

**Problem:** Oracles need decentralization and economic security

**EigenLayer Solution:**

```
Oracle AVS:
- Operators fetch and attest to off-chain data
- Aggregation of multiple data sources
- Slashing for incorrect data submission
- Inherits Ethereum's economic security
```

#### 4. Cross-Chain Bridge

**Problem:** Bridges need secure validation of cross-chain messages

**EigenLayer Solution:**

```
Bridge AVS:
- Operators validate cross-chain state transitions
- Multi-sig with economic security backing
- Slashing for signing invalid bridges
- Fraud proof mechanisms
```

#### 5. Fast Finality Layer

**Problem:** Users want faster than 12-15 minute Ethereum finality

**EigenLayer Solution:**

```
Fast Finality AVS:
- Operators provide pre-confirmation signatures
- Users get instant finality (with restaked security)
- Slashing if pre-confirmations contradicted
- Reverts to Ethereum finality if needed
```

---

## Operators

### Operator Role

**Operators** are the backbone of EigenLayer's security model. They are entities that:

1. Run validation software for one or more AVSs
2. Accept delegations from stakers
3. Earn rewards for providing validation services
4. Face slashing risk for misbehavior

**Operator Types:**

```
Professional Operators:
- Infrastructure companies (e.g., Figment, P2P, Coinbase Cloud)
- Run multiple AVSs at scale
- High-quality infrastructure and monitoring
- Strong reputation and track record

Independent Operators:
- Individual validators or small teams
- May specialize in specific AVSs
- More niche or experimental services
```

### Operator Responsibilities

**1. Infrastructure Management**

Operators must maintain robust infrastructure:

```
Infrastructure Requirements (varies by AVS):
├─> Compute: CPUs, GPUs, memory
├─> Storage: Fast SSDs for state data
├─> Network: High bandwidth, low latency
├─> Monitoring: Alerting and uptime tracking
└─> Security: Key management, DDoS protection
```

**2. Running AVS Software**

Each AVS has specific software requirements:

```bash
# Example: Running an AVS client
./avs-client \
  --operator-key 0x... \
  --eigenlayer-rpc https://... \
  --avs-contract 0x... \
  --monitoring-endpoint ...
```

**3. Key Management**

Operators manage multiple types of keys:

```
Operator Key Types:
├─> ECDSA Key (Ethereum address)
│   └─> Used for on-chain transactions
│   └─> Registering/deregistering from AVSs
│
└─> BLS Key (Aggregatable signatures)
    └─> Used for AVS consensus
    └─> Efficient signature aggregation
    └─> See: /fundamentals/cryptography/bls.md
```

**4. Operator Registration**

```solidity
interface IDelegationManager {
    struct OperatorDetails {
        address earningsReceiver;  // Where rewards are sent
        address delegationApprover; // Optional: who can delegate
        uint32 stakerOptOutWindowBlocks; // Withdrawal delay
    }

    /// @notice Register as an operator
    function registerAsOperator(
        OperatorDetails calldata registeringOperatorDetails,
        string calldata metadataURI
    ) external;

    /// @notice Modify operator details
    function modifyOperatorDetails(
        OperatorDetails calldata newOperatorDetails
    ) external;
}
```

### Delegation System

**How Delegation Works:**

```
Staker → Chooses Operator → Delegates Stake
                │
                ├─> Operator uses stake to opt into AVSs
                │
                └─> Operator and Staker share rewards
                    (minus operator commission)
```

**Delegation Smart Contract:**

```solidity
interface IDelegationManager {
    /// @notice Delegate stake to an operator
    function delegateTo(
        address operator,
        ISignatureUtils.SignatureWithExpiry memory approverSignatureAndExpiry,
        bytes32 approverSalt
    ) external;

    /// @notice Undelegate from current operator
    function undelegate(address staker) external returns (bytes32);

    /// @notice Get delegated shares for an operator
    function operatorShares(
        address operator,
        IStrategy strategy
    ) external view returns (uint256);
}
```

**Delegation Shares Calculation:**

When stakers delegate, their shares contribute to the operator's total delegated stake:

```
Example:
- Alice deposits 10 stETH → Receives 10 shares
- Bob deposits 20 stETH → Receives 20 shares
- Both delegate to Operator Charlie

Charlie's delegated stake:
├─> Strategy: stETH
└─> Total Shares: 30 shares
    └─> Underlying: 30 stETH of economic security
```

**Withdrawal Queue:**

Undelegating requires waiting in a withdrawal queue:

```
Undelegation Flow:
1. Staker calls undelegate()
   └─> Enters withdrawal queue

2. Wait for withdrawal delay (e.g., 7 days)
   └─> Prevents instant exits during slashing events

3. Complete withdrawal
   └─> Receive tokens back

4. Can redelegate to different operator
```

### Operator Economics

**Revenue Streams:**

```
Operator Revenue:
├─> AVS Rewards (variable by AVS)
│   ├─> Per-task payments
│   ├─> Block rewards
│   └─> Token emissions
│
└─> Operator Commission (10-20% typical)
    └─> Taken from delegated staker rewards
```

**Commission Structure:**

```solidity
// Commission is set at the AVS level
interface IAVSServiceManager {
    function setOperatorCommission(uint256 commissionBps) external;
    // commissionBps in basis points (e.g., 1000 = 10%)
}
```

**Example Economics:**

```
Scenario:
- Operator has 1000 ETH delegated
- Validates 3 AVSs
- Each AVS pays 5% APR
- Operator takes 15% commission

Annual Rewards:
├─> Gross AVS Rewards: 1000 ETH × 15% = 150 ETH
├─> Operator Commission: 150 ETH × 15% = 22.5 ETH
└─> Delegator Rewards: 150 ETH × 85% = 127.5 ETH

Operator Total: 22.5 ETH
Delegator APR: 127.5 / 1000 = 12.75%
```

**Reputation and Delegation Market:**

Operators compete for delegations based on:

```
Reputation Factors:
├─> Historical uptime and performance
├─> Number of AVSs validated
├─> Commission rates
├─> Slashing history (or lack thereof)
├─> Brand and trust
└─> Additional services (MEV redistribution, etc.)
```

### Operator Security

**1. Key Management**

Critical security considerations:

```
Key Security Best Practices:
├─> ECDSA Keys (Ethereum):
│   ├─> Hardware wallets for high-value operations
│   ├─> Multi-sig for operator registration
│   └─> Separate hot/cold wallets
│
└─> BLS Keys (AVS consensus):
    ├─> Secure key generation ceremonies
    ├─> Hardware Security Modules (HSMs)
    ├─> Key rotation procedures
    └─> See: /fundamentals/cryptography/bls.md
          (includes BLS rogue key attack mitigation)
```

**2. Multi-AVS Risk Management**

Operators must manage compounding slashing risk:

```
Risk Management Strategies:
├─> Limit number of concurrent AVSs
│   └─> More AVSs = more slashing surface
│
├─> Stake allocation limits
│   └─> Cap exposure to any single AVS
│
├─> AVS due diligence
│   └─> Audit slashing conditions carefully
│
└─> Insurance and hedging
    └─> Slashing insurance protocols (if available)
```

**3. Slashing Exposure Limits**

Operators can set maximum slashing exposure:

```solidity
interface IOperatorSlashingLimits {
    struct SlashingLimit {
        address avs;
        uint256 maxSlashablePercentageBps; // e.g., 1000 = 10%
    }

    /// @notice Set per-AVS slashing limits
    function setSlashingLimits(
        SlashingLimit[] calldata limits
    ) external;
}
```

**4. Monitoring and Alerting**

Essential monitoring:

```
Monitoring Checklist:
├─> AVS client uptime
├─> Network connectivity
├─> Slashing events (on all AVSs)
├─> Delegated stake changes
├─> Reward accrual
├─> Smart contract upgrades
└─> Slashing condition updates
```

---

## Slashing

### Slashing Overview

**Slashing** is the mechanism by which misbehaving operators (and their delegators) lose stake as punishment for protocol violations. It is the core enforcement mechanism that makes restaked economic security meaningful.

**Why Slashing Exists:**

```
Without Slashing:
Operator → Validates incorrectly → No consequence → Security breaks

With Slashing:
Operator → Validates incorrectly → Loses stake → Incentivized to behave correctly
```

**Who Can Be Slashed:**

```
Slashing Affects:
├─> Operator (directly responsible)
└─> Delegated Stakers (share operator's fate)
    └─> Incentivizes delegators to choose good operators
```

**Slashing vs. Jailing:**

```
Jailing (Temporary):
- Operator temporarily removed from active set
- No fund loss
- Can rejoin after timeout
- Used for: Downtime, minor infractions

Slashing (Permanent Fund Loss):
- Operator loses portion of stake
- Stake burned or redistributed
- Used for: Provable misbehavior, malicious actions
```

### Slashing Conditions

Each AVS defines its own slashing conditions based on what constitutes misbehavior in that specific service.

**Common Slashing Conditions:**

#### 1. Safety Violations

```
Examples:
- Signing conflicting messages
- Double-signing blocks
- Attesting to invalid state transitions
```

**Smart Contract Example:**

```solidity
interface IDataAvailabilitySlasher {
    /// @notice Slash operator for unavailable data
    function slashForUnavailability(
        address operator,
        bytes32 blobCommitment,
        bytes calldata unavailabilityProof
    ) external;

    struct UnavailabilityProof {
        bytes32 blobHash;
        uint256 timestamp;
        bytes32[] merkleProof;
        // Proof that data is not available
    }
}
```

#### 2. Liveness Violations

```
Examples:
- Excessive downtime
- Missing assigned tasks
- Unresponsive to queries
```

#### 3. Invalid Attestations

```
Examples:
- Signing incorrect oracle data
- Attesting to non-existent state
- Providing false proofs
```

**Smart Contract Example:**

```solidity
interface IOracleSlasher {
    /// @notice Slash for incorrect data submission
    function slashForIncorrectData(
        address operator,
        bytes32 dataHash,
        bytes calldata correctData,
        bytes calldata proof
    ) external;

    struct IncorrectDataProof {
        bytes32 submittedDataHash;
        bytes correctData;
        bytes signature; // Operator's signature on incorrect data
        uint256 timestamp;
    }
}
```

### Slashing Process

**Standard Slashing Flow:**

```
1. Misbehavior Occurs
   └─> Operator violates slashing condition

2. Challenge/Proof Submitted
   └─> Anyone can submit proof of misbehavior

3. On-Chain Verification
   └─> Smart contract verifies proof validity

4. Slashing Executed (if proof valid)
   └─> Funds deducted from operator and delegators

5. Distribution
   └─> Slashed funds burned or redistributed
```

**Implementation Example:**

```solidity
interface IAVSServiceManager {
    event OperatorSlashed(
        address indexed operator,
        address indexed avs,
        uint256 amount,
        string reason
    );

    struct SlashingRequest {
        address operator;
        uint256 slashAmount;
        string description;
        bytes proof;
        uint256 timestamp;
    }

    /// @notice Submit slashing proof
    function submitSlashingProof(
        SlashingRequest calldata request
    ) external returns (bytes32 slashingId);

    /// @notice Verify and execute slashing
    function executeSlashing(
        bytes32 slashingId
    ) external;

    /// @notice Challenge incorrect slashing
    function challengeSlashing(
        bytes32 slashingId,
        bytes calldata challengeProof
    ) external;
}
```

**Slashing Parameters:**

```solidity
interface ISlashingParameters {
    struct Parameters {
        uint256 maxSlashablePercentageBps;  // Max % of stake slashable
        uint256 challengeWindow;             // Time to challenge slashing
        uint256 withdrawalDelayBlocks;       // Delay before withdrawal
        address slashingAuthority;           // Who can trigger slashing
    }

    function getSlashingParameters(address avs)
        external view returns (Parameters memory);
}
```

### Veto Committee

Some AVSs implement a **Veto Committee** to prevent false positives and malicious slashing.

**Veto Committee Purpose:**

```
Prevents:
├─> False positive slashing (honest operator mistakenly slashed)
├─> Griefing attacks (malicious slashing attempts)
├─> Software bugs causing incorrect slashing
└─> Edge cases not considered in slashing conditions
```

**Veto Mechanism:**

```solidity
interface IVetoCommittee {
    struct VetoProposal {
        bytes32 slashingId;
        uint256 vetoThreshold; // e.g., 5 out of 7 committee members
        uint256 vetoDeadline;
        mapping(address => bool) votes;
        uint256 voteCount;
    }

    /// @notice Committee member votes to veto slashing
    function voteToVeto(bytes32 slashingId) external;

    /// @notice Execute veto if threshold reached
    function executeVeto(bytes32 slashingId) external;
}
```

**Veto Time Window:**

```
Slashing Timeline with Veto:

T=0: Slashing proof submitted
│
├─ T+0 to T+24h: Veto committee review period
│  └─> Committee can veto if false positive
│
└─ T+24h: Veto window expires
   └─> Slashing executed (if not vetoed)
```

**Governance Balance:**

```
Trade-off:
├─> Too much committee power → Centralization risk
├─> Too little → False positives harm honest operators
└─> Ideal: Committee as backstop for edge cases only
```

### Slashing Security Considerations

**1. False Positive Prevention**

Critical to avoid slashing honest operators:

```
Mitigation Strategies:
├─> Rigorous proof requirements
├─> Multiple independent validators must agree
├─> Sufficient proof verification time
├─> Veto committee as backstop
└─> Conservative slashing thresholds initially
```

**2. Griefing Attack Prevention**

Attackers might try to cause nuisance slashing:

```solidity
interface IAntiGriefing {
    /// @notice Require bond to submit slashing proof
    function submitSlashingProofWithBond(
        SlashingRequest calldata request
    ) external payable returns (bytes32);
    // Bond returned if proof valid
    // Bond slashed if proof invalid (griefing attempt)
}
```

**3. Economic Feasibility**

Slashing must be economically meaningful:

```
Attack Cost > Attack Benefit

Example:
- To corrupt oracle: Need 51% of staked capital
- Staked capital: $100M
- Attack cost: $51M (to corrupt majority)
- Bridge exploit value: $40M
- Attack unprofitable → Security maintained
```

**4. Cascading Slashing Risk**

Risk of multiple simultaneous slashings:

```
Cascading Scenario:
1. Operator validates AVS-A, AVS-B, AVS-C
2. Software bug affects all three
3. All three slash simultaneously
4. Total slashing: 30% + 20% + 25% = 75% of stake
5. Catastrophic loss for operator and delegators
```

**Mitigation:**

```solidity
interface ICascadingProtection {
    /// @notice Limit total slashing in a time period
    function setMaxSlashingPerPeriod(
        uint256 maxPercentageBps,
        uint256 periodBlocks
    ) external;
    // E.g., Max 20% slashing per 30-day period
}
```

**5. Front-Running Slashing**

Operators might try to withdraw before slashing executes:

```
Attack:
1. Operator knows they misbehaved
2. Quickly undelegates stake
3. Tries to withdraw before slashing proof submitted

Prevention:
- Withdrawal delays (e.g., 7-14 days)
- Slashing can affect queued withdrawals
- Lookback period for slashing
```

**Implementation:**

```solidity
interface ISlashingLookback {
    /// @notice Slash can affect withdrawals queued within lookback period
    function slashWithLookback(
        address operator,
        uint256 lookbackBlocks
    ) external;
    // Slashing affects stake that was active during lookback period
}
```

---

## Smart Contract Architecture

### Key Contracts

EigenLayer's smart contract system consists of several core contracts that work together:

```
EigenLayer Contract System:

┌─────────────────────────────────────────────────────────┐
│                  Core Contracts                          │
├─────────────────────────────────────────────────────────┤
│                                                           │
│  StrategyManager          DelegationManager              │
│  ┌─────────────────┐     ┌──────────────────┐          │
│  │ - Deposits      │────>│ - Operator reg   │          │
│  │ - Withdrawals   │     │ - Delegation     │          │
│  │ - Strategies    │     │ - Shares mgmt    │          │
│  └─────────────────┘     └──────────────────┘          │
│          │                        │                      │
│          │                        │                      │
│          ▼                        ▼                      │
│  ┌─────────────────────────────────────────┐           │
│  │         AVSDirectory                     │           │
│  │  - AVS registration                      │           │
│  │  - Operator opt-ins                      │           │
│  │  - Service agreements                    │           │
│  └─────────────────────────────────────────┘           │
│                                                           │
│  ┌─────────────────┐     ┌──────────────────┐          │
│  │  EigenPodManager│     │   Slasher        │          │
│  │  - Native stake │     │   - Slashing     │          │
│  │  - EigenPods    │     │   - Freezing     │          │
│  └─────────────────┘     └──────────────────┘          │
│                                                           │
└─────────────────────────────────────────────────────────┘
```

**1. StrategyManager**

Manages deposits, withdrawals, and strategies for LST restaking.

**2. DelegationManager**

Handles operator registration and staker delegation.

**3. AVSDirectory**

Registry of AVSs and operator opt-ins.

**4. EigenPodManager**

Manages native restaking through EigenPods.

**5. Slasher**

Handles slashing logic and operator freezing.

### Contract Interfaces

#### StrategyManager Interface

```solidity
interface IStrategyManager {
    /// @notice Deposits tokens into specified strategy
    function depositIntoStrategy(
        IStrategy strategy,
        IERC20 token,
        uint256 amount
    ) external returns (uint256 shares);

    /// @notice Queue withdrawals from strategies
    function queueWithdrawals(
        QueuedWithdrawalParams[] calldata queuedWithdrawalParams
    ) external returns (bytes32[] memory);

    struct QueuedWithdrawalParams {
        IStrategy[] strategies;
        uint256[] shares;
        address withdrawer;
    }

    /// @notice Complete queued withdrawal
    function completeQueuedWithdrawal(
        Withdrawal calldata withdrawal,
        IERC20[] calldata tokens,
        uint256 middlewareTimesIndex,
        bool receiveAsTokens
    ) external;

    struct Withdrawal {
        address staker;
        address delegatedTo;
        address withdrawer;
        uint256 nonce;
        uint32 startBlock;
        IStrategy[] strategies;
        uint256[] shares;
    }

    /// @notice Get staker's shares in a strategy
    function stakerStrategyShares(
        address staker,
        IStrategy strategy
    ) external view returns (uint256 shares);
}
```

#### DelegationManager Interface

```solidity
interface IDelegationManager {
    struct OperatorDetails {
        address earningsReceiver;
        address delegationApprover;
        uint32 stakerOptOutWindowBlocks;
    }

    /// @notice Register as an operator
    function registerAsOperator(
        OperatorDetails calldata registeringOperatorDetails,
        string calldata metadataURI
    ) external;

    /// @notice Modify operator details
    function modifyOperatorDetails(
        OperatorDetails calldata newOperatorDetails
    ) external;

    /// @notice Delegate to an operator
    function delegateTo(
        address operator,
        ISignatureUtils.SignatureWithExpiry memory approverSignatureAndExpiry,
        bytes32 approverSalt
    ) external;

    /// @notice Undelegate from current operator
    function undelegate(address staker) external returns (bytes32[] memory withdrawalRoot);

    /// @notice Get operator's delegated shares for a strategy
    function operatorShares(
        address operator,
        IStrategy strategy
    ) external view returns (uint256);

    /// @notice Check if address is an operator
    function isOperator(address operator) external view returns (bool);

    /// @notice Get staker's delegated operator
    function delegatedTo(address staker) external view returns (address);
}
```

#### AVSDirectory Interface

```solidity
interface IAVSDirectory {
    /// @notice Register operator to AVS
    function registerOperatorToAVS(
        address operator,
        ISignatureUtils.SignatureWithSaltAndExpiry memory operatorSignature
    ) external;

    /// @notice Deregister operator from AVS
    function deregisterOperatorFromAVS(address operator) external;

    /// @notice Cancel operator's deregistration
    function cancelOperatorDeregistration(address operator) external;

    /// @notice Check if operator is registered to AVS
    function isOperatorRegistered(
        address avs,
        address operator
    ) external view returns (bool);

    /// @notice Update AVS metadata URI
    function updateAVSMetadataURI(string calldata metadataURI) external;

    /// @notice Calculate operator AVS registration digest
    function calculateOperatorAVSRegistrationDigestHash(
        address operator,
        address avs,
        bytes32 salt,
        uint256 expiry
    ) external view returns (bytes32);
}
```

#### EigenPod Interface

```solidity
interface IEigenPod {
    /// @notice Verify withdrawal credentials of validator(s)
    function verifyWithdrawalCredentials(
        uint64 oracleTimestamp,
        BeaconChainProofs.StateRootProof calldata stateRootProof,
        uint40[] calldata validatorIndices,
        bytes[] calldata validatorFieldsProofs,
        bytes32[][] calldata validatorFields
    ) external;

    /// @notice Verify and process beacon chain withdrawals
    function verifyAndProcessWithdrawals(
        uint64 oracleTimestamp,
        BeaconChainProofs.StateRootProof calldata stateRootProof,
        BeaconChainProofs.WithdrawalProof[] calldata withdrawalProofs,
        bytes[] calldata validatorFieldsProofs,
        bytes32[][] calldata validatorFields,
        bytes32[][] calldata withdrawalFields
    ) external;

    /// @notice Withdraw funds from EigenPod
    function withdrawBeforeRestaking() external;

    /// @notice Activate restaking
    function activateRestaking() external;
}
```

#### Strategy Interface

```solidity
interface IStrategy {
    /// @notice Deposit tokens into strategy
    function deposit(
        IERC20 token,
        uint256 amount
    ) external returns (uint256 shares);

    /// @notice Withdraw tokens from strategy
    function withdraw(
        address recipient,
        IERC20 token,
        uint256 amountShares
    ) external;

    /// @notice Convert underlying amount to shares
    function underlyingToShares(
        uint256 amountUnderlying
    ) external view returns (uint256);

    /// @notice Convert shares to underlying amount
    function sharesToUnderlying(
        uint256 amountShares
    ) external view returns (uint256);

    /// @notice Get total shares in strategy
    function totalShares() external view returns (uint256);

    /// @notice Get underlying token
    function underlyingToken() external view returns (IERC20);
}
```

### Contract Interactions

**Full User Flow with Contract Calls:**

#### 1. Deposit and Delegate Flow

```solidity
// Step 1: Approve tokens
IERC20(stETH).approve(address(strategyManager), amount);

// Step 2: Deposit into strategy
uint256 shares = IStrategyManager(strategyManager).depositIntoStrategy(
    IStrategy(stETH_strategy),
    IERC20(stETH),
    amount
);

// Step 3: Delegate to operator
IDelegationManager(delegationManager).delegateTo(
    operatorAddress,
    operatorSignature,
    salt
);
```

#### 2. Operator Registration and AVS Opt-In

```solidity
// Step 1: Register as operator
IDelegationManager.OperatorDetails memory details = IDelegationManager.OperatorDetails({
    earningsReceiver: operatorRewardsAddress,
    delegationApprover: address(0), // No approver needed
    stakerOptOutWindowBlocks: 50400 // ~7 days
});

IDelegationManager(delegationManager).registerAsOperator(
    details,
    "ipfs://metadata-uri"
);

// Step 2: Opt into AVS
IAVSDirectory(avsDirectory).registerOperatorToAVS(
    operatorAddress,
    operatorSignature
);
```

#### 3. Withdrawal Flow

```solidity
// Step 1: Queue withdrawal
IStrategyManager.QueuedWithdrawalParams[] memory params =
    new IStrategyManager.QueuedWithdrawalParams[](1);

params[0] = IStrategyManager.QueuedWithdrawalParams({
    strategies: strategies,
    shares: shares,
    withdrawer: msg.sender
});

bytes32[] memory withdrawalRoots = IStrategyManager(strategyManager).queueWithdrawals(params);

// Step 2: Wait for withdrawal delay...

// Step 3: Complete withdrawal
IStrategyManager(strategyManager).completeQueuedWithdrawal(
    withdrawal,
    tokens,
    middlewareTimesIndex,
    true // receiveAsTokens
);
```

**Contract Interaction Diagram:**

```
User Actions:
─────────────

Restaking Flow:
User → IERC20.approve() → StrategyManager
User → StrategyManager.depositIntoStrategy() → Strategy
User → DelegationManager.delegateTo() → Operator

Operator Actions:
─────────────────

Operator Setup:
Operator → DelegationManager.registerAsOperator()
Operator → AVSDirectory.registerOperatorToAVS() → AVS

AVS Validation:
Operator → [Off-chain AVS validation software]
         → AVS smart contracts (task completion, attestations)

Withdrawal:
───────────

User → StrategyManager.queueWithdrawals()
User → [Wait for withdrawal delay]
User → StrategyManager.completeQueuedWithdrawal()
```

---

## Resources

### Official Documentation

- [EigenLayer Documentation](https://docs.eigenlayer.xyz/) - Official documentation
- [EigenLayer Whitepaper](https://docs.eigenlayer.xyz/overview/whitepaper) - Original research paper
- [EigenLayer GitHub](https://github.com/Layr-Labs) - Core contracts and AVS examples

### Technical Resources

- [EigenLayer Contracts Repository](https://github.com/Layr-Labs/eigenlayer-contracts) - Smart contract source code
- [EigenLayer Middleware](https://github.com/Layr-Labs/eigenlayer-middleware) - AVS development framework
- [AVS Developer Guide](https://docs.eigenlayer.xyz/eigenlayer/avs-guides/avs-developer-guide) - Building AVSs on EigenLayer

### Related Concepts in This Repository

- [BLS Signatures](../../../../fundamentals/cryptography/bls.md) - Used for operator consensus
- [BLS Rogue Key Attack](../../../../Findings/infrastructure/bls-rogue-key-attack.md) - Security considerations
- [Node Operations](../../../../fundamentals/node-operations/) - Validator infrastructure

### Example AVSs

- **EigenDA**: Data availability layer ([GitHub](https://github.com/Layr-Labs/eigenda))
