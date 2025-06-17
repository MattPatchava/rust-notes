# Unicode, and UTF-8, and Base64

### Unicode vs. UTF-8

#### Unicode

Unicode is a standard that **maps characters to unique numbers**. This is necessary because binary systems cannot represent human-readable characters (`A`, `Ж`, `中`, `🔥`, etc.) directly.

It defines:

* **What characters exist**
* **Their (arbitrary but logical) IDs**

#### UTF-8

UTF-8 (Unicode Transformation Format, 8-bit) is an encoding format — it specifies how to store a number (defined by Unicode) in bytes (hence "8-bit").

##### Why UTF-8 is Necessary

A simple number like `6` would be stored as `0x06` (or `0b00000110`), but what happens when you need to store a number like `0x1F525` (the Unicode number for `🔥`)?

Not only does it need to be split across multiple bytes (and there are different ways of doing this), but we need some way of knowing where the byte representation of this character starts and ends in a stream of bytes.

For example, consider the following hypothetical decimal encoding scheme, character `A` is represented by a `1`, character `B` is represented by a `2`, and character `L` is represented by a `12`.

When you encounter `12`, how would you know whether it is representing `AB` or `L`? Without more context (i.e., demarcation of start and end points for each character), you wouldn't know whether the `2` is a continuation of the first character, or the start of a new character. This is where UTF-8 comes in.

UTF-8 uses prefixes to indicate how many bytes are in a given character. For example:

* A single-byte character has a `0` prefix (`0xxxxxxx`)

* The lead byte of a two-byte character is `110xxxxx`

* The lead byte of a three-byte character is `1110xxxx`

  ...and so on

* A continuation byte is `10xxxxxx`

___

### String Representation in Rust vs. Other Languages

* In Rust, strings are stored internally as UTF-8 byte arrays, i.e., arrays of UTF-8 bytes.
  * Since UTF-8 prefixes etc. determine where each character starts and ends, the byte representations themselves are **variable-length**.
  * For example, the character `A` is `0x41` (one byte, since 16 * 16 = 256), while `🔥` is `0xF0 0x9F 0x94 0xA5` (four bytes).
* In some older languages (like C), strings were stored internally as null-terminated byte arrays, where the end of each byte was indicated by a null byte (`\0`).
* In many other languages (like Java), strings are stored as UTF-16 — still variable-length, but less compact than UTF-8.

The reason for this is simply that UTF-8 (and Unicode) were created after C/C++, but before Rust.

* **C**: 1972
* **C++**: 1983
* **Unicode / UTF-8**: 1992
* **Rust**: 2010

___

### Note on Encoding and Decoding

UTF formats (like UTF-8) are considered a type of encoding scheme, so while we have talked about "storing" a Unicode value as bytes, you will commonly hear people say "encode" to UTF-8 — which means exactly the same thing.

___

### Base64

Base64 is an **encoding scheme** (or protocol) used to **convert (encode) binary data to printable ASCII characters**. It is used to **store or transmit binary data over text-based protocols** (e.g., HTTP, JSON, XML).

#### How It Works

1. Take binary data (a stream of bytes)
2. Split it into 3-byte (24-bit) chunks
3. Break each byte chunk into four 6-bit groups
4. Map each 6-bit number to a printable Base64 character
5. Pad with `=` if input length is not divisible by 3

The name **Base64** is chosen because it uses 64 (out of 128 total) ASCII characters to represent binary data. This is because $2^6 = 64$, so Base64 fits nicely into 6-bit chunks.

#### Example

```
ASCII:          D    o    g
Decimal:       68   111  103
Binary:  01000100 01101111 01100111
Split:   010001 000110 111101 100111
Indexes:    17     6     61     39
Base64:     R      G     9      n

"Dog" → "RG9n"
```

#### `base64` Rust Crate

The `base64` Rust crate is used for encoding/decoding between binary and Base64-encoded strings.

* The main two functions you will use from this crate are `encode()` and `decode()`, which are imported as follows:

  ```rust
  use base64::{encode, decode};
  ```

The signatures for the functions are:

##### `encode()`

```rust
fn encode<T: AsRef<[u8]>>(input: T) -> String
```

* **Takes anything that can be viewed as a string slice**, like `&str`, `Vec<u8>`, or `&[u8]`
* **Returns a `String`**, because Base64 is an ASCII-safe text format — so a `String` is the natural choice

##### `decode()`

```rust
fn decode<T: AsRef<[u8]>>(input: T) -> Result<Vec<u8>, DecodeError>
```

* Takes a Base64 string — again, anything that can be viewed as bytes (typically `&str`)
* Returns a `Vec<u8>`, not `[u8]`

The reason that `decode()` returns a `Vec<u8>` instead of an array is that:

* The function doesn't know how long the encoded data will be ahead of time

* `[u8]` is a **slice**, which:

  * Has no ownership
  * Needs to point to some already-allocated memory

* `Vec<u8>` is **owned**, resizable, and heap-allocated, and the function needs to dynamically allocate based on the input
