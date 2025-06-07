# Optional Derive Macros

When you want to implement the `Parser` trait from the `clap` crate using `#[derive(Parser)]`, you import it using the following command:

```bash
cargo add clap --features derive
```

While this is a common import pattern across various Rust crates, it does not seem intuitive at first. This is because what we want to import is the `Parser` trait specifically, and if you haven't learned about optional derive macros, the above command looks like you are trying to import `derive` or something — a built-in Rust keyword, so this doesn't make sense.

___

### Invoking Procedural Macros

First, remember that:

* `#[derive()]` is a Rust pattern/keyword used to invoke **procedural macros**. Since the `derive` keyword is used for this, procedural macros can also be referred to as "derive macros".
* `Parser` is one such procedural/derive macro.

___

### Optional Parts of a Crate and the `--features` Flag

The `--features` flag in Cargo is used to **enable parts of a crate that are not included by default**. These features may:

* Add extra dependencies
* Expose macros or APIs
* Enable specific modules or behaviours

This functionality **allows crates to remain lightweight** by default, while **letting you opt in to more powerful or specialised capabilities**.

___

### `clap_derive` and the `Parser` Trait Implementation

The `Parser` trait is not included in the core `clap` crate. Instead, it is implemented in a separate but related crate called `clap_derive`. This naming convention is a common feature across different crates, as we will discuss later.

To import this crate, you *could* technically call `cargo add clap_derive`, however, this is not the proper approach. Instead, the `clap_derive` crate is part of the public-facing interface of `clap`, which means it can be imported along with the core `clap` crate using `--features derive`.

That is, even though the `Parser` trait is defined in `clap_derive`, which is technically a separate crate, the `clap` crate re-exports this trait when the `derive` feature is enabled. So, in your code, you still do `use clap::Parser`, not `use clap_derive::Parser`.

Using `--features derive` is preferred over independently importing `clap_derive` because it allows the maintainers of `clap` to:

* Potentially rename `clap_derive`
* Swap it out
* Add setup glue

If you directly added `clap_derive`, then any change to its internals would break lots of downstream crates.

___

### This is a Common Pattern in Rust

In libraries that offer optional derive macros, this is the standard approach:

| Crate       | Features to Enable Derive                                  |
| ----------- | ---------------------------------------------------------- |
| `serde`     | `derive` → enables `serde_derive`                          |
| `clap`      | `derive` → enables `clap_derive`                           |
| `thiserror` | `default` → enables `thiserror-impl`                       |
| `tokio`     | `macros` → enables procedural macros like `#[tokio::main]` |

___

### How It Works Internally (Important for Understanding)

The `clap` crate contains the following:

```rust
#[cfg(feature = "derive")]
pub use clap_derive::Parser;
```

* The `cfg` attribute stands for "configuration", but more specifically, it refers to **conditional compilation configuration**.

  So, `#[cfg(feature = "derive")]` means:

  * ***"Only compile this block if the current compilation configuration includes the feature named `derive`."***

* The `cfg` attribute is **compile-time** and removes code entirely from the final binary if the condition is not met.

This means that regardless of whether you call `cargo add clap --features derive` or just `cargo add clap`, the Rust source code is identical. However, the compiler attributes result in different compiled binaries depending on what configuration parameters were passed to them at compile time.

Therefore, **the exact same Rust source code can compile to significantly different binaries depending on the compiler configuration**.

This feature is somewhat unique to Rust and not so common in other languages.

#### Extra Background for Deeper Understanding

The question I had that ultimately led to this was:

* *"If you import the `clap` crate without also importing the `clap_derive` crate, then given that it is the core `clap` crate that conditionally imports and exposes `clap_derive`", if you tried to `use clap::Parser` in your source code, the error would ultimately originate from the `clap` source code failing to import `Parser` from `clap_derive`, right?"*

The answer is no, and as we now understand, the reason for this is that **the inclusion of `clap_derive` is done at compile time, not at runtime**.

#### Rust Is More Tightly Coupled With Its Package Manager than Other Languages

Because `cargo` is used to pass arguments to compiler attributes like `cfg`, it is more tightly coupled with Rust than say, `npm` is with Node.js or `pip` is with Python.

To elaborate:

* Feature flags are a first-class part of Cargo and `Cargo.toml`:
  * They are not ad hoc conventions, they are deeply woven into how the crate is built and what gets compiled.
* The compiler directly respects Cargo metadata:
  * The compiler knows which features are enabled because Cargo passes them in via `--cfg feature="xyz"` flags at build-time.
* Using dependencies, enabling features, and customising builds **only happens through Cargo**.
  * There is no concept of a "manual build" in Rust.
  * However, it is important to remember that Cargo does not replace the Rust compiler (`rustc`), but configures it and passes flags to it like `rustc --cfg feature="derive"`.

The coupling between Rust and Cargo is deliberate. Its purpose is to provide consistency, safety, and ease of reproducibility.