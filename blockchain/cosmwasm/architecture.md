# CosmWasm Architecture

CosmWasm is a smart contract platform built for the Cosmos ecosystem, allowing developers to write secure, multi-chain smart contracts in Rust compiled to WebAssembly (Wasm).

## Core Concepts

### Contract Lifecycle

CosmWasm contracts follow a three-phase lifecycle:

1. **Instantiate** - Initialize the contract with state
2. **Execute** - Handle state-changing operations
3. **Query** - Read-only queries of contract state

## Entry Points

Every CosmWasm contract must implement three main entry points:

### 1. Instantiate

Called once when the contract is first deployed to the blockchain.

```rust
#[cfg_attr(not(feature = "library"), entry_point)]
pub fn instantiate(
    deps: DepsMut,
    env: Env,
    info: MessageInfo,
    msg: InstantiateMsg,
) -> Result<Response, ContractError> {
    // Initialize contract state
    let state = State {
        owner: info.sender.clone(),
        count: msg.count,
    };
    STATE.save(deps.storage, &state)?;

    Ok(Response::new()
        .add_attribute("method", "instantiate")
        .add_attribute("owner", info.sender)
        .add_attribute("count", msg.count.to_string()))
}
```

**Parameters:**
- `deps: DepsMut` - Mutable dependencies (storage, API access, querier)
- `env: Env` - Environment info (block height, time, contract address)
- `info: MessageInfo` - Message metadata (sender, funds sent)
- `msg: InstantiateMsg` - Custom initialization parameters

### 2. Execute

Handles all state-changing operations after instantiation.

```rust
#[cfg_attr(not(feature = "library"), entry_point)]
pub fn execute(
    deps: DepsMut,
    env: Env,
    info: MessageInfo,
    msg: ExecuteMsg,
) -> Result<Response, ContractError> {
    match msg {
        ExecuteMsg::Increment {} => execute_increment(deps),
        ExecuteMsg::Reset { count } => execute_reset(deps, info, count),
        ExecuteMsg::Transfer { recipient, amount } => {
            execute_transfer(deps, env, info, recipient, amount)
        }
    }
}

fn execute_increment(deps: DepsMut) -> Result<Response, ContractError> {
    STATE.update(deps.storage, |mut state| -> Result<_, ContractError> {
        state.count += 1;
        Ok(state)
    })?;

    Ok(Response::new().add_attribute("action", "increment"))
}
```

### 3. Query

Read-only operations that don't modify state.

```rust
#[cfg_attr(not(feature = "library"), entry_point)]
pub fn query(deps: Deps, _env: Env, msg: QueryMsg) -> StdResult<Binary> {
    match msg {
        QueryMsg::GetCount {} => to_binary(&query_count(deps)?),
        QueryMsg::GetOwner {} => to_binary(&query_owner(deps)?),
    }
}

fn query_count(deps: Deps) -> StdResult<CountResponse> {
    let state = STATE.load(deps.storage)?;
    Ok(CountResponse { count: state.count })
}
```

**Key Difference:** Uses `Deps` (immutable) instead of `DepsMut`.

## Cross-Contract Communication

CosmWasm uses **messages** for cross-contract calls, following an actor model where contracts communicate asynchronously.

### Message Types

#### 1. CosmosMsg - Execute actions on other contracts

```rust
use cosmwasm_std::{CosmosMsg, WasmMsg, to_binary};

pub fn execute_call_other_contract(
    deps: DepsMut,
    other_contract: String,
) -> Result<Response, ContractError> {
    // Prepare message for another contract
    let msg = ExecuteMsg::SomeAction {
        param: "value".to_string()
    };

    // Create cross-contract call
    let wasm_msg = WasmMsg::Execute {
        contract_addr: other_contract.clone(),
        msg: to_binary(&msg)?,
        funds: vec![], // Can send tokens with the call
    };

    Ok(Response::new()
        .add_message(wasm_msg)
        .add_attribute("action", "call_other_contract")
        .add_attribute("target", other_contract))
}
```

#### 2. SubMsg - Execute with reply handling

SubMsgs allow you to handle the response from cross-contract calls:

```rust
use cosmwasm_std::{SubMsg, ReplyOn};

const REPLY_ID: u64 = 1;

pub fn execute_with_reply(
    deps: DepsMut,
    other_contract: String,
) -> Result<Response, ContractError> {
    let msg = WasmMsg::Execute {
        contract_addr: other_contract,
        msg: to_binary(&ExecuteMsg::SomeAction {})?,
        funds: vec![],
    };

    // Use SubMsg to get a callback
    let sub_msg = SubMsg {
        id: REPLY_ID,
        msg: msg.into(),
        gas_limit: None,
        reply_on: ReplyOn::Success, // or ReplyOn::Error, ReplyOn::Always
    };

    Ok(Response::new().add_submessage(sub_msg))
}

#[cfg_attr(not(feature = "library"), entry_point)]
pub fn reply(deps: DepsMut, _env: Env, msg: Reply) -> Result<Response, ContractError> {
    match msg.id {
        REPLY_ID => {
            // Handle the reply from the cross-contract call
            let response = msg.result.into_result().map_err(StdError::generic_err)?;

            // Access return data
            if let Some(data) = response.data {
                // Process the response
            }

            Ok(Response::new().add_attribute("reply", "success"))
        }
        _ => Err(ContractError::UnknownReplyId { id: msg.id }),
    }
}
```

#### 3. WasmQuery - Query other contracts

```rust
use cosmwasm_std::{WasmQuery, QueryRequest};

pub fn query_other_contract(
    deps: Deps,
    other_contract: String,
) -> StdResult<CountResponse> {
    let query_msg = QueryMsg::GetCount {};

    let query = WasmQuery::Smart {
        contract_addr: other_contract,
        msg: to_binary(&query_msg)?,
    };

    let response: CountResponse = deps.querier.query(&QueryRequest::Wasm(query))?;
    Ok(response)
}
```

## Message Flow Examples

### Simple Cross-Contract Call

```
Contract A                 Contract B
    |                          |
    |  Execute(msg) -------->  |
    |                          | (processes)
    |  <-------- Response      |
    |                          |
   Done
```

### SubMsg with Reply

```
Contract A                 Contract B
    |                          |
    |  SubMsg(id=1) -------->  |
    |                          | (processes)
    |  <-- Success/Error       |
    |                          |
  reply(id=1)
    | (handle response)
   Done
```

### Chained Execution

```rust
pub fn execute_chain(deps: DepsMut) -> Result<Response, ContractError> {
    // Multiple messages executed in order
    let msg1 = WasmMsg::Execute {
        contract_addr: "contract1".to_string(),
        msg: to_binary(&ExecuteMsg::Action1 {})?,
        funds: vec![],
    };

    let msg2 = WasmMsg::Execute {
        contract_addr: "contract2".to_string(),
        msg: to_binary(&ExecuteMsg::Action2 {})?,
        funds: vec![],
    };

    Ok(Response::new()
        .add_message(msg1)  // Executed first
        .add_message(msg2)) // Executed second
}
```

## Security Considerations

### 1. Message Ordering

Messages in a Response are executed **sequentially**. If any message fails, the entire transaction reverts.

```rust
// If msg2 fails, msg1 is also reverted
Ok(Response::new()
    .add_message(msg1)
    .add_message(msg2)) // Transaction fails here -> all reverted
```

### 2. Reentrancy Protection

CosmWasm is **reentrancy-safe by default** because:
- State changes are committed only after the entire transaction succeeds
- No callbacks during execution (unlike Ethereum)
- The actor model prevents direct reentrancy

### 3. Reply Handling

Always validate reply data and handle errors:

```rust
pub fn reply(deps: DepsMut, _env: Env, msg: Reply) -> Result<Response, ContractError> {
    match msg.result {
        SubMsgResult::Ok(response) => {
            // Validate response data before using
            if let Some(data) = response.data {
                let parsed: MyData = from_binary(&data)?;
                // Validate parsed data
                ensure!(parsed.value > 0, ContractError::InvalidReply);
            }
            Ok(Response::new())
        }
        SubMsgResult::Err(err) => {
            // Handle error appropriately
            Err(ContractError::SubMsgFailed { reason: err })
        }
    }
}
```

### 4. Authorization

Always verify the message sender:

```rust
pub fn execute_admin_action(
    deps: DepsMut,
    info: MessageInfo,
) -> Result<Response, ContractError> {
    let state = STATE.load(deps.storage)?;

    // Check authorization
    if info.sender != state.owner {
        return Err(ContractError::Unauthorized {});
    }

    // Proceed with admin action
    Ok(Response::new())
}
```

## Common Patterns

### 1. Factory Pattern

```rust
pub fn execute_create_instance(
    deps: DepsMut,
    env: Env,
    code_id: u64,
) -> Result<Response, ContractError> {
    let instantiate_msg = WasmMsg::Instantiate {
        admin: Some(env.contract.address.to_string()),
        code_id,
        msg: to_binary(&InstantiateMsg {})?,
        funds: vec![],
        label: "New Instance".to_string(),
    };

    let sub_msg = SubMsg {
        id: INSTANTIATE_REPLY_ID,
        msg: instantiate_msg.into(),
        gas_limit: None,
        reply_on: ReplyOn::Success,
    };

    Ok(Response::new().add_submessage(sub_msg))
}
```

### 2. Callback Pattern

```rust
// Contract A calls Contract B, B calls back to A
pub fn execute_callback(
    deps: DepsMut,
    info: MessageInfo,
    data: String,
) -> Result<Response, ContractError> {
    // Verify callback came from expected contract
    let config = CONFIG.load(deps.storage)?;
    ensure!(
        info.sender == config.trusted_contract,
        ContractError::Unauthorized {}
    );

    // Process callback data
    process_callback_data(deps, data)?;

    Ok(Response::new().add_attribute("action", "callback_received"))
}
```

## Resources

- [CosmWasm Documentation](https://docs.cosmwasm.com/)
- [CosmWasm Book](https://book.cosmwasm.com/)
- [CosmWasm GitHub](https://github.com/CosmWasm/cosmwasm)
- [CosmWasm Plus (Standard Contracts)](https://github.com/CosmWasm/cw-plus)
