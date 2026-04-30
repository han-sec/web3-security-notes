# Diamond Proxy (EIP-2535)

### Overview:

The Diamond Proxy pattern (EIP-2535) is an upgradeable contract pattern that allows a single proxy contract to delegate calls to multiple implementation contracts called **facets**. Each facet handles a subset of functions, identified by their function selectors. This solves the 24KB contract size limit and enables granular upgradeability.

### Key Components

1. **Diamond (Proxy Contract)** — The single entry point that delegates calls to facets.
2. **Facets (Implementation Contracts)** — Multiple contracts, each containing a group of related functions.
3. **DiamondCut Facet** — Special facet that manages adding, replacing, or removing facets.
4. **DiamondLoupe Facet** — Introspection facet that reports which facets and functions exist.

### Core Concept

The Diamond maintains a mapping of function selectors to facet addresses. When a call arrives, the Diamond looks up which facet handles that selector and delegates the call to it. All facets share the Diamond's storage via `delegatecall`.

```
User Call (selector 0xabcd1234)
  -> Diamond Proxy
  -> Lookup selector in mapping
  -> delegatecall to Facet B (which owns that selector)
  -> Executes using Diamond's storage
```

### Technical Details

### 1. Diamond Storage

Facets share the Diamond's storage, so a structured storage pattern is used to avoid collisions:

```solidity
library LibDiamond {
    bytes32 constant DIAMOND_STORAGE_POSITION = keccak256("diamond.standard.diamond.storage");

    struct FacetAddressAndPosition {
        address facetAddress;
        uint96 functionSelectorPosition;
    }

    struct DiamondStorage {
        // selector => facet address and position in selectors array
        mapping(bytes4 => FacetAddressAndPosition) selectorToFacetAndPosition;
        // facet addresses
        mapping(address => uint256) facetFunctionSelectors;
        // owner of the diamond
        address contractOwner;
    }

    function diamondStorage() internal pure returns (DiamondStorage storage ds) {
        bytes32 position = DIAMOND_STORAGE_POSITION;
        assembly {
            ds.slot := position
        }
    }
}
```

Each facet can also define its own namespaced storage using the same `keccak256` slot pattern, known as **Diamond Storage** or **AppStorage**.

### 2. DiamondCut — Adding, Replacing, Removing Functions

The `diamondCut` function is how facets are managed:

```solidity
interface IDiamondCut {
    enum FacetCutAction { Add, Replace, Remove }

    struct FacetCut {
        address facetAddress;
        FacetCutAction action;
        bytes4[] functionSelectors;
    }

    function diamondCut(
        FacetCut[] calldata _diamondCut,
        address _init,
        bytes calldata _calldata
    ) external;
}
```

- **Add**: Map new selectors to a new facet address.
- **Replace**: Change existing selectors to point to a different facet.
- **Remove**: Delete selectors from the mapping (set facet address to `address(0)`).
- **`_init` + `_calldata`**: Optional initialization call after the cut (like an initializer).

### 3. DiamondLoupe — Introspection

Required by EIP-2535 for transparency:

```solidity
interface IDiamondLoupe {
    struct Facet {
        address facetAddress;
        bytes4[] functionSelectors;
    }

    function facets() external view returns (Facet[] memory);
    function facetFunctionSelectors(address _facet) external view returns (bytes4[] memory);
    function facetAddresses() external view returns (address[] memory);
    function facetAddress(bytes4 _functionSelector) external view returns (address);
}
```

### 4. Fallback Function

The Diamond's fallback does the selector lookup and delegation:

```solidity
fallback() external payable {
    LibDiamond.DiamondStorage storage ds = LibDiamond.diamondStorage();
    address facet = ds.selectorToFacetAndPosition[msg.sig].facetAddress;
    require(facet != address(0), "Diamond: Function does not exist");

    assembly {
        calldatacopy(0, 0, calldatasize())
        let result := delegatecall(gas(), facet, 0, calldatasize(), 0, 0)
        returndatacopy(0, 0, returndatasize())
        switch result
        case 0 { revert(0, returndatasize()) }
        default { return(0, returndatasize()) }
    }
}
```

## Technical Implications

1. **No Contract Size Limit**:
    - Since logic is split across facets, no single contract needs to stay under 24KB.
2. **Granular Upgradeability**:
    - Individual functions can be added, replaced, or removed without touching other facets.
3. **Shared Storage**:
    - All facets operate on the same storage via `delegatecall`. Storage layout must be carefully coordinated.
4. **Gas Costs**:
    - Extra SLOAD for the selector-to-facet lookup on every call. Slightly more than Transparent Proxy overhead.
5. **EIP-165 Support**:
    - Diamonds should implement ERC-165 (`supportsInterface`) and report support for `IDiamondCut` and `IDiamondLoupe`.

## Implementation Considerations

1. **Storage Collisions**:
    - Facets share storage. Use Diamond Storage (namespaced `keccak256` slots) or the AppStorage pattern to avoid collisions between facets.
2. **Selector Conflicts**:
    - Two facets cannot register the same function selector. The `diamondCut` function should revert on conflicts.
3. **Initialization**:
    - Use the `_init` address and `_calldata` in `diamondCut` for initializing new facets, similar to `initializer` in other proxy patterns.
4. **Access Control**:
    - `diamondCut` must be protected. Typically only the owner or a governance contract can call it.
5. **Immutable Functions**:
    - Functions can be made "immutable" by adding them directly to the Diamond contract itself (not as a facet), so they cannot be replaced via `diamondCut`.

## Advantages

1. No 24KB contract size limit — logic is split across many facets.
2. Granular upgrades — add, replace, or remove individual functions.
3. Single address for the entire protocol — simpler for users and integrations.
4. On-chain introspection via DiamondLoupe.
5. Can organize related functions into logical facets for better code organization.

## Disadvantages

1. Significantly more complex than Transparent Proxy or UUPS.
2. Storage management across facets is error-prone.
3. Harder to audit — reviewers must understand the full facet graph and storage layout.
4. Higher gas per call due to selector lookup (extra SLOAD).
5. Tooling support is weaker compared to OpenZeppelin's Transparent/UUPS proxies.

## Usage Example

```solidity
// Deploy facets
DiamondCutFacet cutFacet = new DiamondCutFacet();
DiamondLoupeFacet loupeFacet = new DiamondLoupeFacet();
ERC20Facet erc20Facet = new ERC20Facet();

// Prepare cuts
IDiamondCut.FacetCut[] memory cuts = new IDiamondCut.FacetCut[](2);

cuts[0] = IDiamondCut.FacetCut({
    facetAddress: address(loupeFacet),
    action: IDiamondCut.FacetCutAction.Add,
    functionSelectors: getLoupeSelectors()
});

cuts[1] = IDiamondCut.FacetCut({
    facetAddress: address(erc20Facet),
    action: IDiamondCut.FacetCutAction.Add,
    functionSelectors: getERC20Selectors()
});

// Deploy Diamond with initial cuts
Diamond diamond = new Diamond(owner, address(cutFacet));
IDiamondCut(address(diamond)).diamondCut(cuts, initAddress, initCalldata);

// Interact — all calls go through the single Diamond address
IERC20(address(diamond)).transfer(to, amount);
```

## Security Considerations

1. **Protect `diamondCut`** — If an attacker gains access, they can replace any function with malicious logic. Use multisig or governance.
2. **Storage layout audits** — Every facet upgrade must be checked for storage slot collisions with existing facets.
3. **Selector clashing** — Verify no two facets register the same selector. Tooling like Louper can help.
4. **Facet removal risks** — Removing a facet deletes its functions but not its storage. Leftover storage can cause issues if a new facet reuses those slots.
5. **Delegatecall risks** — A malicious or buggy facet has full access to the Diamond's storage and can corrupt state for all other facets.
6. **Initialization replay** — Ensure facet initializers cannot be called twice (use initialized flags in Diamond Storage).

## Comparison with Other Proxy Patterns

| Feature | Transparent Proxy | UUPS | Diamond |
|---------|------------------|------|---------|
| Implementation contracts | 1 | 1 | Many (facets) |
| Max contract size | 24KB | 24KB | Unlimited (split across facets) |
| Upgrade granularity | Entire implementation | Entire implementation | Per-function |
| Gas overhead | Admin check | Minimal | Selector lookup (SLOAD) |
| Complexity | Medium | Low | High |
| On-chain introspection | No | No | Yes (DiamondLoupe) |
