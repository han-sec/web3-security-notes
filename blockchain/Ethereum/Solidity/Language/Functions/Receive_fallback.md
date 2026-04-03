## Differences between `receive()` and `fallback()` functions in detail:

**receive() function:**

```javascript
// Can only have this exact signature
receive() external payable {
    // code
}
```

Used when:

- ETH is sent to contract (msg.value > 0)
- NO calldata (empty calldata)
- Must be marked payable

**fallback() function:**

```javascript
// Can have either signature:
fallback() external [payable] {
    // code
}
// OR
fallback(bytes calldata input) external [payable] returns (bytes memory output) {
    // code
}
```

**Used when:**

- No other function matches the function identifier OR
- When ETH is sent and no receive() exists
- Payable is optional

**Control Flow for Incoming Transactions:**

```javascript
Transaction Received
       |
       ↓
Is msg.data empty?
       |
       ├── Yes → Is msg.value > 0?
       |          |
       |          ├── Yes → receive() exists?
       |          |          ├── Yes → receive()
       |          |          └── No → fallback() exists?
       |          |                    ├── Yes and payable → fallback()
       |          |                    └── No or not payable → revert
       |          |
       |          └── No → receive() exists?
       |                    ├── Yes → receive()
       |                    └── No → fallback() exists?
       |                              ├── Yes → fallback()
       |                              └── No → revert
       |
       └── No → Matching function exists?
                ├── Yes → Execute function
                └── No → fallback() exists?
                          ├── Yes → fallback()
                          └── No → revert
```

Examples:

```javascript
// Example contract
contract Example {
    event ReceivedETH(uint value);
    event FallbackCalled(bytes data, uint value);

    // Called when ETH sent with empty calldata
    receive() external payable {
        emit ReceivedETH(msg.value);
    }

    // Called when:
    // 1. Function doesn't exist
    // 2. ETH sent but no receive()
    fallback() external payable {
        emit FallbackCalled(msg.data, msg.value);
    }
}
```

```javascript
// Usage scenarios:
address(contract).transfer(1 ether);  // calls receive()
address(contract).send(1 ether);      // calls receive()
address(contract).call{value: 1 ether}("");  // calls receive()
address(contract).call{value: 1 ether}("0x123456");  // calls fallback()
address(contract).call("nonexistentFunction()");  // calls fallback()
```

**Key Differences:**

```javascript
receive():
- Must be payable
- Cannot have arguments
- Cannot return anything
- Only for empty calldata with ETH

fallback():
- Optionally payable
- Can have input/output (in newer versions)
- Handles any non-matching calls
- Backup for ETH transfers if no receive()
```

**Common Use Cases:**

```javascript
// Simple ETH receiver
contract SimpleReceiver {
    receive() external payable {
        // Handle ETH receipt
    }
}

// Smart contract wallet
contract SmartWallet {
    receive() external payable {
        // Log ETH deposits
    }
    
    fallback() external payable {
        // Delegate all other calls
        // to another contract
    }
}

// Proxy contract
contract Proxy {
    address implementation;
    
    fallback() external payable {
        // Forward all calls to implementation
        (bool success, ) = implementation.delegatecall(msg.data);
        require(success);
    }
}
```

**Gas Considerations:**

- `receive()` typically has 2300 gas when called via transfer/send
- `fallback()` has no such restriction when called directly
- Both get full gas when called via `call()`