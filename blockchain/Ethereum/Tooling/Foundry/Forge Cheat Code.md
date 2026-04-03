Common forge cheat code for testing:

1. Event Testing:

```solidity
// vm.expectEmit - Test event emissions
vm.expectEmit(checkTopic1, checkTopic2, checkTopic3, checkData);
// Parameters:
// - checkTopic1: Check first indexed parameter
// - checkTopic2: Check second indexed parameter
// - checkTopic3: Check third indexed parameter
// - checkData: Check non-indexed parameters

// Example:
function testEventEmission() public {
		// 1. Tell foundry what event to expect
    vm.expectEmit(true, true, false, true);
		// 2. Emit the expected event
    emit Transfer(alice, bob, 100);
		// 3. Make the actual call
    token.transfer(bob, 100);
}
```

1. Revert Testing:

```solidity
// vm.expectRevert - Test function reverts
vm.expectRevert();// Any revert
vm.expectRevert("error message"); // Specific error message

// Specific error selector
vm.expectRevert(bytes4);
vm.expectRevert(abi.encodeWithSelector(CustomError.selector, params));

// Example:
function testRevert() public {
    vm.expectRevert("Insufficient balance");
    token.transfer(bob, tooMuchAmount);

// Custom errors
    vm.expectRevert(abi.encodeWithSelector(
        CustomError.InsufficientBalance.selector,
        balance,
        amount
    ));
}

```

1. Call Testing:

```solidity
// vm.expectCall - Test external calls
vm.expectCall(
    address target,// Contract being called
    uint256 value,// ETH value (optional)
    bytes calldata data// Encoded function call
);

// Example:
function testTransfer() public {
    vm.expectCall(
        address(token),
        abi.encodeCall(IERC20.transfer, (recipient, 100))
    );
    contract.withdraw(100);
}

```

1. State Manipulation:

```solidity
// Time
vm.warp(timestamp);// Set block timestamp
vm.roll(blockNum);// Set block number// Account state

// Msg.sender
vm.prank(address);// Set msg.sender for next call
vm.startPrank(address);// Set msg.sender for all subsequent calls
vm.stopPrank();// Stop prank

// Dealing funds
hoax(address, uint) // vm.prank + deal
vm.deal(address, uint);// Set ETH balance

// Example:
function testTimeDependent() public {
    vm.warp(startTime + 1 days);
    vm.prank(alice);
    contract.withdraw();
}
```

1. Storage/Memory:

```solidity
// Storage
vm.store(address, bytes32 slot, bytes32 value);// Set storage
vm.load(address, bytes32 slot);// Read storage

// Memory recording
vm.record();

// Start recording storage reads/writes
(bytes32[] memory reads, bytes32[] memory writes) = vm.accesses(address);
```

1. Environment:

```solidity
// Chain
vm.chainId(uint256);// Set chainId
vm.fee(uint256);// Set tx.gasprice

// Account
vm.etch(address, bytes code);// Set contract code
vm.label(address, string);// Label address for traces
```

1. Mocking:

```solidity
// Mock calls
vm.mockCall(
    address,
    bytes calldata,
    bytes memory
);// Mock return value
vm.clearMockedCalls();

// Example:
function testMock() public {
    vm.mockCall(
        address(token),
        abi.encodeWithSelector(IERC20.balanceOf.selector, user),
        abi.encode(1000)
    );
}
```

1. Assertions:

```solidity
// Fuzzing
vm.assume(bool);// Assume condition for fuzzing// Gas
vm.pauseGasMetering();
vm.resumeGasMetering();
```

1. Comparison/Assertion:

```solidity
// Value Matching
assertEq(a, b);     // Assert equal (uint, int, address)
assertEq(a, b, "message");  // With custom error message
assertNotEq(a, b);  // Assert not equal

// Boolean Checks
assertTrue(condition);    // Assert true
assertFalse(condition);   // Assert false

// Approximate Equality (for floating point)
assertApproxEqAbs(a, b, maxDelta);  // Within absolute difference
assertApproxEqRel(a, b, maxPercentDelta);  // Within percentage

// Array/Bytes Comparison
assertEq(bytes1, bytes2);  // Compare byte arrays
assertEq(array1, array2);  // Compare arrays
```

Common Patterns:

```solidity
// Testing payable functions
hoax(address, uint256);// Combination of deal + prank

// Testing time windows
skip(uint256);// Advance time
rewind(uint256);// Reverse time// Sign messages
vm.sign(uint256 privateKey, bytes32 digest)
```