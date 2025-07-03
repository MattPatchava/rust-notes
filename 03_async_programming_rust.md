# Asynchronous Programming

This is a comprehensive guide to all the various concepts related to asynchronous programming, with a focus on its implementation in Rust.

### Core Concepts

#### 1. Async (Asynchronous Programming) — General Term

* **Definition**: A programming model where operations (usually I/O) are started, and the program continues running without waiting for them to complete — instead, it "awaits" a notification when they are ready.
* **Key Points**:
  * **Non-Blocking**: Execution continues instead of waiting.
  * **Single-Threaded or Multi-Threaded**: Async **does not require threads**, but works well with or without them.
  * **Runtime**: Driven by a **runtime** (like `tokio` or `actix-rt`) that schedules and polls tasks. This is **similar to the JavaScript event loop**.
  * Typically used for I/O-bound tasks (network, file, DB).
* **Think: "Efficient waiting."**

#### 2. Multithreading

* **Definition**: A programming model where a program spawns multiple **threads of execution**, potentially running in parallel across multiple CPU cores.

* **Key Points**:

  * Threads have their **own call stack and execution path**.
  * Can be preemptive (OS-controlled) or cooperative (like green threads).
  * Threads can run truly in parallel if hardware supports it (multiple cores).
  * Ideal for CPU-bound tasks (e.g., parallel computations).
  * While each thread has its own stack, threads do **share memory with other threads in the same process**.

* **Think**: This is like multiple things that need to be done, but rather than doing one at a time, you switch between them, i.e., do a bit of the first task, then a bit of the second, then back to the first, then a bit of the third, etc.

  If you have a friend (or multiple) with you, then the tasks are truly being executed in parallel, but each friend (CPU core) may still be working asynchronous (i.e., while you keep switching between tasks 1-3, your friend keeps switching between tasks 4-6).

#### 3. Concurrency

* **Definition**: Another general term like "asynchronous programming". "Concurrency" simply refers to doing multiple things at once (asynchronously), and can be used to refer to multithreading, parallelism, async runtimes, etc.

#### 4. Parallelism

* **Definition**: Running **multiple tasks simultaneously**, literally at the same time.
* **Key Points**:
  * Requires multiple CPU cores
  * Best for CPU-bound tasks (e.g., image processing, simulations)
  * Can be implemented with threads or distributed systems
* **Think**: Actual code running side by side.

#### 5. CPU Cores and Their Role

* A CPU core can execute one thread at a time.
* More cores → more **true parallelism** possible.
* Async programs may still run on a single core, but handle many concurrent I/O tasks efficiently.
* Multithreaded programs can fully utilise multiple cores.

___

### Async Runtimes vs. Kernel Scheduling

#### Overview

The fundamental difference between an async runtime and the kernel scheduler is that a **kernel scheduler controls when and which thread runs on a CPU core**, while a user-space **async runtime (e.g., `tokio`) controls what happens within a thread** (e.g., which async task or future to poll next).

#### Async Runtimes

An async runtime is as **user-space construct** — a library (not part of the OS) that manages and drives asynchronous tasks in your program.

In Rust, examples include:

* `tokio` — full-featured, multithreaded async runtime
* `async-std` — similar to Node-style runtime
* `actix-rt` — runtime used by Actix

The async runtime:

* Manages tasks (futures) and wakes them when they are ready to make progress.
* Often includes:
  * **Reactor**: Listens for I/O readiness
  * **Executor**: Runs the polled futures
  * Optional **thread pool** (Tokio's multi-thread runtime)
* Is **fully in user-space**: the OS is not aware of it; it is just your program doing clever coordination.

#### Kernel Scheduling

The OS **kernel scheduler manages threads and processes at a system level, across all programs on your machine**.

It:

* Decides which thread runs on which core at which time.
* Uses preemptive multitasking — forcibly pausing threads to switch to others.
* Is concerned with CPU time, not "whether an I/O task is ready".
* Knows nothing about your futures or async tasks — only actual threads and syscalls.

#### Interaction Between Async Runtime and Kernel Scheduling

| Feature                 | Async Runtime (e.g., Tokio) | Kernel Scheduler                     |
| ----------------------- | --------------------------- | ------------------------------------ |
| Operates In             | User space                  | Kernel space                         |
| Schedules               | Futures (Rust async tasks)  | Threads (OS-level execution units)   |
| Preemptive?             | Usually **cooperative**     | Always **preemptive**                |
| Scope of Visibility     | Your app's async logic      | All system threads                   |
| Handles I/O Readiness?  | Yes                         | Indirectly (via syscalls + blocking) |
| Works with async/await? | Yes                         | No — handles lower-level scheduling  |

___

### Futures (Rust-Specific)

#### Overview

* A `Future` in Rust is **any type that implements the `Future` trait**.
* The `Future` trait is **implemented by various types that handle data that is fetched asynchronously**, such as data coming in on a network socket or I/O peripheral, etc.
* Specifically, a `Future` instance **represents a computation that may of may not have completed yet**. The `Future` trait **defines how an asynchronous computation progresses**.

A `Future` in Rust is **conceptually similar to a JavaScript Promise**, though quite different in implementation.

#### Syntax

* `async fn fetch_data()` is just syntactic sugar for `fn fetch_data() -> impl Future<Output = String>`.

  That is, `async fn` is sugar for a function that **returns an anonymous type implementing `Future`**.

* The compiler generates a unique struct to represent the `Future` — you never name it yourself, which is why the return type `impl Future` is used rather than an explicit type.

* At first glance, this looks like dynamic dispatch, however, the compiler knows the exact return type, it just hides it for convenience and abstraction. Dynamic dispatch would instead look like `Box<dyn Future<Output = String>>`.

* The generic `Output` parameter specifies the data type that will ultimately be unwrapped from the `Future` once the `Future` is ready.

#### Types Implementing `Future`

The various types that implement `Future` all have their own internal implementations, such as a file descriptor or socket handle, buffer fields, etc.

##### Internal State Machine

Types implementing `Future` will typically have an internal state machine field, often represented by an enum with states like `Ready` or `Pending`. The use of a state machine is a **convention**, but is not enforced. First, traits only declare methods (behaviour), not fields. And second, different types of Futures may require different state management, so this is completely up to the developer.

When the Future is first instantiated, its state machine will typically be set to a `Pending` state (or equivalent).

#### Calling an Async Function (`.poll()`, `Waker`, Syscalls, and the Task Queue)

##### 1. Function Call

* When you call an async function, it instantly returns a `Future`. The body of the function does not execute at all yet.
* Execution of the function body only begins when the `Future`'s `.poll()` method is called.

##### 2. Initial Polling of the Future

You do not typically call the `Future`'s `.poll()` method directly, this is done by the async runtime. This can be initiated in either of the following two ways:

* Call `.await` on the `Future`

  ```rust
  let fut = fetch_data();
  fut.await;
  ```

* Pass the `Future` to `tokio::spawn()` (if the return value of the async function is not required)

  ```rust
  tokio::spawn(fetch_data));
  ```

##### 3. Runtime Handling of the `Future`

In response to you calling `.await` or `tokio::spawn()`, Tokio:

* Instantiates a `Task` type
* Assigns the `Future` to an internal field of this `Task`
* Generates a `Waker` instance (which is not stored in the `Task` but will be passed to the `Future`'s `.poll()` method)

**The `Waker` is like a notification system for when the `Future` is made ready.**

*Note: The `Waker` is wrapped in a `Context` struct, this is just Rust ergonomics and not conceptually important.*

**Next, Tokio calls the `.poll()` method on the `Future`, passing it the `Waker` (wrapped in a `Context` instance)**.

This finally gets the body of the async function to execute. Somewhere in the async function body (typically a nested function call), a syscall (e.g., `epoll`, `kqueue`, etc.) is made to the kernel, returning a file descriptor.

##### 4. The Awaited Data is Made Available and the Kernel Notifies the Async Runtime

* In response to the syscall, the kernel says: "socket is readable" (or similar).

  Tokio is watching the epoll/kqueue file descriptor, and when it sees that the data has become available, it calls the `.wake()` method on the `Waker`.

* The `.wake()` method adds the `Task` to Tokio's task queue, from which it is eventually dequeued and `.poll()` is called on the `Future` again. This time, since the `Future` is ready, it returns `Ready(val)` and its data is finally made available to the application.

##### Additional Note: Why `Waker`s are Used

You might be wondering why a `Waker` is required at all, as it seems redundant compared to just having the `.wake()` method on the `Task` struct directly. There are several reasons for this, including:

* The `Waker` API is a standardised interface defined in the Rust standard library, but each runtime (Tokio, async-std, smol, etc.) builds its own custom implementations of the internal wake-up logic behind the scenes.
* Using a `Waker` field provides access to the required methods without exposing the entire `Task` instance.
* `Waker`s often need to be cloned and are cheaper to clone than an entire `Task`.
