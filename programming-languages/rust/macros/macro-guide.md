# Macros in Rust

## Naming

| Term                    | Example            | Type      | Description                                      |
| ----------------------- | ------------------ | --------- | ------------------------------------------------ |
| **Attribute**           | `#[derive(Debug)]` | Attribute | Compiler directive attached to an item           |
| **Derive Macro**        | `Debug`            | Macro     | Code generator invoked by `derive`               |
| **Attribute Macro**     | `#[my_macro]`      | Macro     | Custom procedural macro attached directly        |
| **Function-like Macro** | `println!()`       | Macro     | Like function calls but expanded at compile-time |
| **Declarative Macro**   | `macro_rules!`     | Macro     | Pattern-based code generator                     |

Perfect — let’s break this down **step by step** so you get the full mental model. ✅

---

### 🧩 In Rust, there are **two main categories** of macros

#### 1. **Declarative Macros**

* Also known as **"macro_rules!" macros**.
* Defined using `macro_rules!` keyword.
* These are **pattern-based** — they match syntax patterns and expand into code.
* They’re kind of like *advanced find-and-replace with syntax awareness*.
* **Examples:**

  ```rust
  macro_rules! say_hello {
      () => {
          println!("Hello!");
      };
  }

  fn main() {
      say_hello!(); // expands to println!("Hello!");
  }
  ```

* 📦 Used heavily in the standard library (`vec!`, `println!`, `format!`, etc).

---

#### 2. **Procedural Macros**

* These are **functions** that take **Rust syntax (token stream)** as input and **generate code**.
* Defined inside a **proc-macro crate** with `proc-macro` type.
* Much more **powerful** — you can manipulate syntax trees programmatically.

Now, **Procedural Macros** are divided into **3 subtypes**:

---

### ⚙️ Three Types of Procedural Macros

#### (a) **Custom Derive Macro**

* Triggered with `#[derive(MyTrait)]`
* Automatically implements a trait for a struct or enum.
* Example:

  ```rust
  // in proc-macro crate
  #[proc_macro_derive(MyTrait)]
  pub fn my_trait_derive(input: TokenStream) -> TokenStream {
      // parse input, generate impl MyTrait for struct
  }
  ```

* **Example use:**

  ```rust
  #[derive(MyTrait)]
  struct Data;
  ```

* ✅ Most common — e.g. `#[derive(Debug, Clone, Serialize)]`.

---

#### (b) **Attribute-like Macro**

* Looks like a **custom attribute**: `#[some_attr]`
* More general than `derive` — can be placed on **functions, modules, etc.**
* Example:

  ```rust
  #[route(GET, "/")]
  fn index() {}
  ```

* Implemented like:

  ```rust
  #[proc_macro_attribute]
  pub fn route(attr: TokenStream, item: TokenStream) -> TokenStream {
      // transform the function
  }
  ```

---

#### (c) **Function-like Macro**

* Looks like a **function call**, but invoked as a macro:

  ```rust
  my_macro!(input_tokens);
  ```

* Implemented like:

  ```rust
  #[proc_macro]
  pub fn my_macro(input: TokenStream) -> TokenStream {
      // return generated code
  }
  ```

* Similar syntax to declarative macros, but with **full control over parsing**.

---

### 🔹 Summary Table

| Type                           | Syntax Example              | Defined Using             | Input Type     | Typical Use Case                      |
| ------------------------------ | --------------------------- | ------------------------- | -------------- | ------------------------------------- |
| **Declarative Macro**          | `macro_rules! name { ... }` | `macro_rules!`            | Token patterns | `println!`, `vec!`, `format!`         |
| **Derive Procedural Macro**    | `#[derive(MyTrait)]`        | `#[proc_macro_derive]`    | TokenStream    | Auto trait implementation             |
| **Attribute Procedural Macro** | `#[route(GET, "/")]`        | `#[proc_macro_attribute]` | 2 TokenStreams | Custom function/struct transformation |
| **Function-like Macro**        | `my_macro!(...)`            | `#[proc_macro]`           | TokenStream    | DSL-like expansions                   |

---

✅ `#[derive(Debug)]` → a **derive procedural macro** (not just a normal attribute).
✅ `#[cfg(...)]`, `#[no_mangle]`, `#[inline]` → **built-in attributes**.
✅ `macro_rules!` → **declarative macro** (not procedural).

## Commonly used attribute in Rust

| Attribute                         | Purpose                                                             |
| --------------------------------- | ------------------------------------------------------------------- |
| `#[allow(lint_name)]`             | Suppress a specific warning (e.g., `unreachable_code`, `dead_code`) |
| `#[deny(lint_name)]`              | Turn a lint into a compiler **error**                               |
| `#[warn(lint_name)]`              | Ensure a specific lint shows a warning (even if normally off)       |
| `#[cfg(...)]`                     | Configuration. Conditionally include code based on features, OS, etc.              |
| `#[cfg_attr(...)]`                | Apply an attribute conditionally                                    |
| `#[derive(...)]`                  | Automatically implement traits like `Debug`, `Clone`, etc.          |
| `#[repr(...)]`                    | Control memory layout (e.g., `C`, `packed`, `transparent`)          |
| `#[inline]` / `#[inline(always)]` | Hint to the compiler to inline a function                           |
| `#[test]`                         | Marks a function as a test                                          |
| `#[macro_export]`                 | Expose a macro to other crates                                      |
| `#[no_std]`                       | Compile without the standard library                                |
