# 🧠 Rust Memory Model

Understanding Rust’s memory model helps you reason about **ownership**, **borrowing**, and **safety** — the pillars of Rust’s design.
This note walks through how memory is structured, managed, and protected in a typical Rust program.

---

## 📘 1. Virtual Memory Overview

Every program in a modern operating system runs inside its own **virtual address space**, isolated from other processes.

This isolation is achieved by the **MMU (Memory Management Unit)**, which translates **virtual addresses** (what your program uses) into **physical addresses** (actual RAM).

---

### 🧱 Virtual Memory Concepts

* **Virtual Address Space**
  A continuous address range (e.g. 0x0000… to 0xFFFFFFFF for 32-bit).
  The program “sees” this as if it owns all the memory.

* **Pages**
  Memory is divided into **pages** (commonly 4 KB each).
  The OS manages memory in **page tables** — mapping virtual pages to physical frames.

* **Page Permissions**
  Each page has flags:

  * `r` → readable
  * `w` → writable
  * `x` → executable

  Example:
  Code pages are `rx` (read-execute) but not `w` — prevents self-modifying code and many exploits.

* **Kernel vs User Space**

  * **User space**: Where your Rust program runs.
  * **Kernel space**: Protected area; only OS code can access it.

---

### 🗺️ Typical Layout of a Process

Here’s how the virtual memory space is commonly structured:

```
High Addresses
      ↓
+-----------------------+
| Kernel space          | <- inaccessible to user programs
+-----------------------+
| Stack                 | <- grows downward
|                       |
|                       |
|         ↓             |
+-----------------------+
| Heap                  | <- grows upward
|         ↑             |
|                       |
|                       |
+-----------------------+
| Data segment          |
|   - .data (initialized globals)
|   - .bss  (uninitialized globals)
+-----------------------+
| Text / Code segment   | <- compiled machine instructions (read-only)
+-----------------------+
Low Addresses
```

---

## 🧩 2. Memory Regions in Rust

Rust uses these memory regions just like C/C++, but enforces **safety guarantees** at compile-time.

| Region          | Description                   | Example                      |
| --------------- | ----------------------------- | ---------------------------- |
| **Text / Code** | Compiled machine code         | `fn main() {}` instructions  |
| **.data**       | Initialized globals/statics   | `static X: i32 = 5;`         |
| **.bss**        | Uninitialized globals/statics | `static mut Y: i32 = 0;`     |
| **Stack**       | Function frames, local vars   | `let x = 10;`                |
| **Heap**        | Dynamically allocated memory  | `Box::new(10)`, `Vec::new()` |

---

### 🧱 Stack

* Allocated **automatically** on function call.
* Freed **automatically** when the function returns.
* Very fast (LIFO order).
* Size is limited (typically 8 MB).

Example:

```rust
fn main() {
    let x = 5; // stored on stack
    let y = 10; // stored on stack
}
```

---

### 📦 Heap

* Allocated **dynamically** (manual request).
* Freed when the **owner** goes out of scope.
* Managed by **Rust’s ownership system**, not a garbage collector.

Example:

```rust
fn main() {
    let b = Box::new(5); // allocate 5 on heap
    println!("{}", b);
} // `b` is dropped here, heap memory is freed
```

---

## 🧮 3. Memory of a Running Rust Program

When a program runs, each function call creates a **stack frame**:

* Stores local variables
* Stores return address
* Stores saved registers

Rust ensures:

* **Stack references** never outlive their scope.
* **Heap allocations** are freed when owners go out of scope.

Example:

```rust
fn greet() {
    let name = String::from("Han"); // heap allocation for "Han"
    println!("Hello, {}", name);
} // name dropped here, heap memory freed
```

---

## ⚠️ 4. Memory Bugs (Formalism)

In low-level languages (like C/C++), common memory bugs include:

| Bug                  | Description                        |
| -------------------- | ---------------------------------- |
| **Use-After-Free**   | Accessing memory after it’s freed  |
| **Double Free**      | Freeing memory twice               |
| **Dangling Pointer** | Pointer to invalid memory          |
| **Buffer Overflow**  | Writing beyond allocated memory    |
| **Data Race**        | Simultaneous unsynchronized access |

Rust’s **ownership**, **borrowing**, and **lifetimes** eliminate these **statically** (at compile time).

---

## 🧱 5. Unsafe Rust (Manual Memory)

Sometimes, safe Rust cannot express a low-level operation (like calling C code or writing device drivers).
That’s when `unsafe` is needed.

```rust
unsafe {
    let mut x: i32 = 10;
    let r = &mut x as *mut i32; // raw pointer
    *r = 20; // dereference raw pointer
    println!("{}", *r);
}
```

In `unsafe` blocks, Rust:

* **Trusts you** to ensure memory safety manually.
* Does **not disable ownership or lifetime rules** — only allows bypassing checks where necessary.

---

### ✅ When Unsafe Is Used Correctly

1. **FFI (Foreign Function Interface)** — Calling C code:

```rust
extern "C" {
    fn printf(fmt: *const u8, ...) -> i32;
}
```

2. **Manual Memory Management** — Building abstractions like `Vec`, `Box`.

3. **Low-level Systems Programming** — Device drivers, OS kernels.

4. **Unsafe Traits / APIs** — Implementing traits like `Send`, `Sync` correctly.

---

## 🧰 Summary

| Concept   | Rust’s Guarantee                         |
| --------- | ---------------------------------------- |
| Ownership | Every value has one owner                |
| Borrowing | Shared or mutable references, never both |
| Lifetimes | Ensure references are valid              |
| Stack     | Fast, auto cleanup                       |
| Heap      | Owned allocations, freed on drop         |
| Unsafe    | Manual memory control (use with care)    |

---
