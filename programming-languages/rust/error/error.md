# Error handling in Rust

## Ways to handle Result

## 1. The `?` operator

Think of `?` as syntactic sugar for a match on `Result`.

This code:

```rust
let contents = std::fs::read_to_string("config.txt")?;
```

Expands roughly to:

```rust
let contents = match std::fs::read_to_string("config.txt") {
    Ok(val) => val,
    Err(e) => return Err(e.into()), // early return!
};
```

Note: The `?` operator is used when we don’t need custom error handling at that spot.

## 2 `map_err`

The method `map_err` exists on `Result<T, E>` and lets you transform the `Err(E)` into another error type.

```rust
impl<T, E> Result<T, E> {
    pub fn map_err<F, O>(self, op: O) -> Result<T, F>
    where
        O: FnOnce(E) -> F,
}
```

## 3 `unwrap`

The method `unwrap` exists on `Result<T, E>`.

```rust
impl<T, E: std::fmt::Debug> Result<T, E> {
    pub fn unwrap(self) -> T
}
```

Behavior:

- If self is `Ok(t)` → returns t.

- If self is `Err(e)` → the program **panics** and **crashes**, printing the error.
