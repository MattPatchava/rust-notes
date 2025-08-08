# `std::fmt`

The `std::fmt` module in Rust's standard library is Rust's **core formatting and printing framework**.

## Printing

Printing means **sending text (usually strings) to an output sink**, typically:

* `stdout` (standard output, e.g., your terminal)
* `stderr`
* A file
* A network socket
* A log system (e.g., `syslog` or `journalctl`)
* An embedded system's display
* Etc.

> #### What is a Sink?
> **A "sink" is just a destination where data is sent**. You can think of it like:
>
> *"A place where data flows into, not out of."*
>
> * A **source** produces data (like your code or a sensor).
>
> * A **sink** consumes data (like stdout, a file, or a log buffer).
>##### Sink vs. Output Stream
> 
> An output stream includes both:
>
> * Sinks (endpoints)
>* Intermediaries (like buffers or pipes)
> 
> So all sinks are part of output streams, but not all output streams are final sinks.

___

## Formatting

Formatting means **transforming a value into a string representation, based on formatting rules**.

It includes:

* Converting numbers into decimal, hexadecimal, binary, etc.

* Handling padding/alignment (e.g., `"{:>8}"` means right-align in 8 characters)

* Limiting precision (e.g., `"Pi is {:.2}"`)

* **Converting custom types into readable strings — that is, defining how your own structs or enums should appear when printed**.

___

### Tools in `std::fmt` for "Formatting" Types as Strings

##### Traits (like `Display`, `Debug`, etc.)

These are formatting traits that types can implement to control how they are formatted when passed into macros like `println!()`, `format!()`, etc.

* `Display` — for user-facing output. Implement this to define how your type is printed with `{}`.
* `Debug` — for developer-facing debugging output. Used with `{:?}`.
* `LowerHex`, `UpperHex`, `Binary`, `Octal`, etc. — for formatting numbers in various bases like hexadecimal, binary, etc. Used with `{:x}`, `{:X}`, `{:b}`, and `{:o}` respectively.

***Each of these traits has a `fmt` method that takes a `Formatter` and writes output to it.***

##### The `Formatter` Struct — Manages Alignment, Precision, etc.

`Formatter` is a helper struct that is passed into `fmt()` method implementations. It holds:

* Width and precision settings

* Alignment flags

* Other formatting configurations

* A reference to an underlying *writer\**, which the `Formatter` writes the formatted string into.

  This may be a:

  * `&mut String` (for `format!()`)
  * `&mut std::io::Stdout` (for `println!()`)
  * Whatever you pass in (for `write!()`)

  *\*A "**writer**" is anything that can be written to. For example, a `String`, a file, a network socket, etc. This feels confusing at first because it doesn't reflect the real-world use of the term "writer". In real life, if you are writing in an exercise book, you are the writer. But in Rust, the book is called the writer.*

  *The reason for this is that for something (like a `String`, `std::fs::File`, etc.) to be able to be written to, it must implement either `std::fmt::Write` (if you are writing strings to it), or `std::io::Write` (if you are writing binary data to it).*

##### Trait Implementations (for standard types like `i32`, `String`, `Option<T>`, etc.)

The standard library provides built-in implementations of formatting traits for common types.

* Numbers (`i32`, `f64`, etc.) implement `Display`, `Debug`, and various numberic formatting traits.
* Containers (like `Option<T>`, `Result<T, E>`, `Vec<T>`, and even tuples implement `Debug`, and sometimes `Display`),
* These trait implementations allow you to print these types using `println!()` or `format!()` without needing to write custom `impl`s for them yourself.

##### Trait Method Conflicts

Let's say a type implements both `Display` and `Debug`. Since both of these have a `.fmt()` method, if you call this method on a type that implements both traits, it is ambiguous which implementation you want to call — the implementation for `Display`, or for `Debug`.

There are two main ways this is resolved (note that these apply to trait conflicts in general, not just `Display` and `Debug`):

* **Only bringing the required traits into scope**:

  * Trait methods are not available unless the trait is in scope.

  * So, if you had:

    * A custom type (say, `Polyglot`)
    * That implemented two traits (`English` and `Swedish`)
    * Both of which had a `.say_hello()` method

    Then:

    * If you only brought `English` into scope, you could call `.say_hello()` on the instance and it would print "Hello".
    * If you only brought `Swedish` into scope, you could call `.say_hello()` on the instance and it would print "Hej".

* **Explicit disambiguation**:

  * Say you had brought both traits into scope:

    ```rust
    use geniuses::Polyglot;
    use languages::{English, Swedish};
    
    fn main() {
        let polyglot: Polyglot = Polyglot::new();
        
        // polyglot.say_hello(); ❌ This would cause a Compiler error: ambiguous
        
        // ✅ Correct way of calling .say_hello()
        English::say_hello(&polyglot);
        Swedish::say_hello(&polyglot);
    }
    ```

___

### Implementing the `std::fmt::Display` Trait

Rust does not assume that your type knows how to format itself as a printable string unless you explicitly tell it how. This is done by implementing traits like `Display` or `Debug`.

Implementing these traits allows you to control things like what fields to show, what format they appear in, etc., when they are passed to a macro like `println!()` to print them to the console.

#### fmt() Method

* **The `Display` trait declares the `fmt()` method, which you must define to implement the `Display` trait.**

* You never typically call this method directly, this is done by formatting macros (`println!()`, `format!()`, etc.) when they format a type with `{}` (Display) or `{:?}` (Debug).

* The signature of the `fmt()` method is:

  ```rust
  fn fmt(&self, f: &mut Formatter<'_>) -> fmt::Result;
  ```

  ##### Parameters:

  * `&self`: The instance you are formatting.
  * `f: &mut Formatter<'_>`: A mutable reference to a `std::fmt::Formatter`, which holds a reference to the writer (target to be written into, implementing `std::fmt::Write`) and formatting settings (like alignment, width, etc.).

  ##### Return Type:

  * `fmt::Result`: This is **an alias for `Result<(), std::fmt::Error>`**. You must return `Ok(())` if successful, otherwise `Err(fmt::Error)`.

##### Execution Flow

The execution flow is like this:

1. Pass an instance of the type implementing `Display` into a formatting macro like `println!()`.

2. The formatting macro then:

   * Initiates a `Formatter` instance for formatting the output.

   * Calls `.fmt()` on the type implementing `Display`, passing it the `Formatter` it just created.

   * `.fmt()` must write to the `Formatter` using a macro like `write!()`, passing the `Formatter` instance as the destination for the data to be written.

     This macro (`write!()`) expands to a call that writes into the writer held by the `Formatter`, ensuring that your input is sent to the correct destination (e.g., `stdout`, a `String`, a file, etc.).

#### Why Does the `Formatter` Require an Explicit Lifetime?

One of the `Formatter`'s internal fields (specifically the writer field) is borrowed, not owned by the `Formatter`.

Due to this, you must explicitly annotate the `Formatter` as having a lifetime (even though literally everything does), not because the compiler or anyone else ever thought it didn't, but because **annotating it explicitly with a lifetime enforces rules about the way it can be used**.

Specifically:

* It cannot be stored in a variable
* It cannot be returned
* It cannot outlive its borrowed writer
* It cannot be kept in a struct unless that struct has the same lifetime

If Rust allowed `Formatter` without a lifetime, then you could accidentally write code like this:

```rust
fn bad<'a>(f: Formatter<'a>) -> &'a mut dyn Write {
    f.writer
}
```

* This would allow you to take ownership of the writer, even though `Formatter` only borrowed it.

* This would cause a memory safety bug.

So, explicitly annotating the `Formatter` with a lifetime is not just saying "this has a lifetime", it is telling the compiler "enforce these memory-safety rules".

#### Real-World Example

When writing a `clap` interface in which the user specifies a hashing protocol in `argv`, rather than just taking a string, we can use an `enum` to handle different hashing protocol variants.

```rust
#[derive(ValueEnum, Clone)]
enum HashingProtocol {
    Sha256,
    Md5,
}
```

However, for `clap` to print information about the parameter that instantiates and determines the variant of this enum, it needs to be able to print information about the enum, so the enum must implement `std::fmt::Display`.

```rust
impl std::fmt::Display for HashingProtocol {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        write!(
            f,
            "{}",
            match self {
                HashingProtocol::Sha256 => "sha256",
                HashingProtocol::Md5 => "md5",
            }
        )
    }
}
```

The `write!()` macro formats a string and writes the formatted string to the writer (implementing `std::fmt::Write`) field in the `Formatter` (`f`).

The `write!()` macro also returns a `Result<(), std::fmt::Error>`, so its return value is returned from `fmt()`.

So, when you pass an instance of `HashingProtocol` to `println!()`, it will:

1. Instantiate a `Formatter`
2. Call `.fmt()` on your `HashingProtocol` instance, passing it the `Formatter` as an argument
3. `fmt()` will call `write!()`, which will:
   * Take an `&str` and format it.
   * Write this to the `Formatter`'s writer (in this case, stdout).
   * Return a `Result<(), std::fmt::Error>`, which will be returned by the outer `fmt` call.
