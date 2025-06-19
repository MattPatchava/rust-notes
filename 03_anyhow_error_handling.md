# `anyhow` Crate

### Background

Consider the following scenario: you're prototyping a Java library and just want to get the core functionality working without getting sidetracked implementing complex error types. You know that a function is fallible, so it has to throw some kind of exception, but that's as far as you want to go with error types for now.

So, you name the exception type for clarity and context, inherit from the `Exception` superclass, and throw it where needed.

```java
public class CustomException extends Exception {
    public CustomException() {
        super();
    }
    
    // Literally not going to do anything else right now
}
```

Now, imagine you want to do the same thing in Rust.

* First, Rust doesn't have OOP-style inheritance, so you cannot simply inherit from a superclass like Java's `Exception`.
* If you make a custom error type, as a bare minimum you will need to manually implement the `std::error::Error` trait for the type — and there is no default implementation.
* You would probably also want to implement the `Display` and `Debug` traits as well.

Clearly this is a lot more work than just throwing a named extension of `Exception` in Java. So, a lot of devs don't do this thoroughly for every error type.

### Prototyping Crates in Rust

Developers of various crates, including many of the most popular ones like `argon2` and `serde_json` were like "yeah we need an error type, but we're not going to implement standard error traits for it, not our problem lol". This isn't laziness, just prioritising the most important features — no one is passionate about errors and we're not writing code for pacemakers or self-driving cars in Rust just yet.

So, this caused the problem of having error types that then needed to be coerced or converted into types that the compiler accepted as error types. For example, say you needed to handle an `argon2::Error` (which does not implement `std::error::Error`), it would fail even if the function returned `Box<dyn std::error::Error>`. You would need to use a method like `.map_err(|e| (Box::new(e) as Box dyn<std::error::Error>))`, which is not only verbose, but likely to lead to new errors.

### The Solution — An Error Wrapper Type (`anyhow::Error`)

In order to solve this problem in a way that can be applied to any error types, no matter how badly they may be slapped together, the `anyhow` crate was created, along with an `anyhow::Error` type, which is essentially a wrapper for any type that is intended to be used for errors, with default implementations for error traits like `Error`, `Display`, and `Debug`.

#### How to Use `anyhow`

The first step is to import the `anyhow::Result` enum type.

```rust
use anyhow::Result;
```

##### Types Implementing `std::error::Error`

When using `anyhow`, it is common practice to make the return type of fallible functions `anyhow::Result<T>`.

For example:

```rust
fn fallible_fn() -> Result<()> {}
```

`Result<T>` is simply shorthand for `Result<T, anyhow::Error>`.

Therefore, if this is your return type, then you can use the `?` operator on any functions that implement all of the following:

* `std::error::Error`
* `'static`
* `Send`
* `Sync`

Using the `?` operator will automatically convert the error to an instance of `anyhow::Error` and return it in the outer function's `Result<T, anyhow::Error>`.

##### All Other Types (Not Implementing `std::error::Error`)

When handling types that don't implement `std::error::Error` (such as `argon2::Error`), using `?` does not automatically coerce them in to `anyhow::Error`. In these cases, you will typically use one of the two methods below to wrap them in an `anyhow::Error` instance and return it:

* `.map_err(|e| anyhow::anyhow!(e))?;`
* `.context("Additional error message here")?;`

The first method `.map_err()` simply takes the `Err` variant of a `Result<T, E>` in a closure and converts it to whatever you return from the method (in the above example, `anyhow::Error`, which is returned by the `anyhow!()` macro). This method itself is not specific to `anyhow`, but can be used to create an `anyhow::Error` to return in the `Result` enum.

The second method `.context()` is specific to `anyhow`. It allows you to pass a string containing additional error message information. It returns an `anyhow::Error` instance containing your custom error message, as well as the information from the error type you passed into it. This method is usually preferred, because it makes you include more context/information, unless the error is obvious.

*Note that it is common practice to `use anyhow::anyhow` so that you can simply use the `anyhow!()` functional macro without the namespace prefix. So, your import will most likely look like the one below:*

```rust
use anyhow::{Result, anyhow};
```

___

### Future Learning

#### `thiserror`

If you later decide to move from prototyping and want to define your own error types while keeping boilerplate low, the `thiserror` crate is often used alongside `anyhow`. `thiserror` helps you define ergonomic custom error types that integrate nicely with `anyhow::Error` via automatic `From` implementations.

#### `anyhow` is used for application-level error handling

`anyhow` is designed for application-level error handling, where you do not need the consumer to match on precise error types. If you are writing a library (crate), prefer designing structured error types (e.g., `thiserror`), so that downstream users can handle errors programatically.