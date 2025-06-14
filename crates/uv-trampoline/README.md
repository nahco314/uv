# Windows trampolines

This is a fork of
[posy trampolines](https://github.com/njsmith/posy/tree/dda22e6f90f5fefa339b869dd2bbe107f5b48448/src/trampolines/windows-trampolines/posy-trampoline).

## Building

### Cross-compiling from Linux

Install [cargo xwin](https://github.com/rust-cross/cargo-xwin). Use your package manager to install
LLD and add the `rustup` targets:

```shell
sudo apt install llvm clang lld
cargo install cargo-xwin
rustup toolchain install nightly-2025-02-16
rustup component add rust-src --toolchain nightly-2025-02-16-x86_64-unknown-linux-gnu
rustup target add --toolchain nightly-2025-02-16 i686-pc-windows-msvc
rustup target add --toolchain nightly-2025-02-16 x86_64-pc-windows-msvc
rustup target add --toolchain nightly-2025-02-16 aarch64-pc-windows-msvc
```

Then, build the trampolines for all supported architectures:

```shell
cargo +nightly-2025-02-16 xwin build --xwin-arch x86 --release --target i686-pc-windows-msvc
cargo +nightly-2025-02-16 xwin build --release --target x86_64-pc-windows-msvc
cargo +nightly-2025-02-16 xwin build --release --target aarch64-pc-windows-msvc
```

### Cross-compiling from macOS

Install [cargo xwin](https://github.com/rust-cross/cargo-xwin). Use your package manager to install
LLVM and add the `rustup` targets:

```shell
brew install llvm
cargo install cargo-xwin
rustup toolchain install nightly-2025-02-16
rustup component add rust-src --toolchain nightly-2025-02-16-aarch64-apple-darwin
rustup target add --toolchain nightly-2025-02-16 i686-pc-windows-msvc
rustup target add --toolchain nightly-2025-02-16 x86_64-pc-windows-msvc
rustup target add --toolchain nightly-2025-02-16 aarch64-pc-windows-msvc
```

Then, build the trampolines for all supported architectures:

```shell
cargo +nightly-2025-02-16 xwin build --release --target i686-pc-windows-msvc
cargo +nightly-2025-02-16 xwin build --release --target x86_64-pc-windows-msvc
cargo +nightly-2025-02-16 xwin build --release --target aarch64-pc-windows-msvc
```

### Updating the prebuilt executables

After building the trampolines for all supported architectures:

```shell
cp target/aarch64-pc-windows-msvc/release/uv-trampoline-console.exe trampolines/uv-trampoline-aarch64-console.exe
cp target/aarch64-pc-windows-msvc/release/uv-trampoline-gui.exe trampolines/uv-trampoline-aarch64-gui.exe
cp target/x86_64-pc-windows-msvc/release/uv-trampoline-console.exe trampolines/uv-trampoline-x86_64-console.exe
cp target/x86_64-pc-windows-msvc/release/uv-trampoline-gui.exe trampolines/uv-trampoline-x86_64-gui.exe
cp target/i686-pc-windows-msvc/release/uv-trampoline-console.exe trampolines/uv-trampoline-i686-console.exe
cp target/i686-pc-windows-msvc/release/uv-trampoline-gui.exe trampolines/uv-trampoline-i686-gui.exe
```

### Testing the trampolines

To perform a basic smoke test of the trampolines, run the following commands on a Windows machine,
from the root of the repository:

```shell
cargo clean
cargo run venv
cargo run pip install black
.venv\Scripts\black --version
```

## Background

### What is this?

Sometimes you want to run a tool on Windows that's written in Python, like `black` or `mypy` or
`jupyter` or whatever. But, Windows does not know how to run Python files! It knows how to run
`.exe` files. So we need to somehow convert our Python file a `.exe` file.

That's what this does: it's a generic "trampoline" that lets us generate custom `.exe`s for
arbitrary Python usage on Windows. You can use it in two ways:

1. **Script runner variant (UVSC)**: bundle a Python script (as a ZIP) alongside the trampoline and
   invoke `python <the bundled script>` at runtime.
2. **Python mirror variant (UVPY)**: create a simple executable that just forwards to a specified
   `python.exe` binary, acting like a stand-in or alias for Python itself.

### How do you use it as Python mirror?

Basically, this invokes the provided `python.exe` executable.

The intended use is:

- First, place our prebuilt `.exe` content at the top of the file.
- After the exe file content, write the path to the Python executable that the script uses to run
  the Python script as UTF-8 encoded string, followed by the path's length as a 32-bit little-endian
  integer.
- Write the magic number 'UVPY' in bytes.

|       `launcher.exe`        |
| :-------------------------: |
|   `<path to python.exe>`    |
| `<len(path to python.exe)>` |
| `<b'U', b'V', b'P', b'Y'>`  |

When you run `python` on this `.exe`, it will see the `UVPY` marker at the end of the file and
recognize that it's the “Python mirror” variant. Then it simply hands everything off to your
specified `python.exe`. Easy-peasy.

### How do you use it as Script runner?

Basically, this looks up `python.exe` (for console programs) and invokes
`python.exe path\to\the\<the .exe>`.

The intended use is:

- First, place our prebuilt `.exe` content at the top of the file.
- After the exe file content, write the path to the Python executable that the script uses to run
  the Python script as UTF-8 encoded string, followed by the path's length as a 32-bit little-endian
  integer.
- Write the magic number 'UVSC' in bytes.
- Finally, rename your Python script as `__main__.py`, compress it into a `.zip` file, and append
  this `.zip` file to the end of one of our prebuilt `.exe` files.

|       `launcher.exe`        |
| :-------------------------: |
|   `<path to python.exe>`    |
| `<len(path to python.exe)>` |
| `<b'U', b'V', b'S', b'C'>`  |
|  `<zipped python script>`   |

Then when you run `python` on the `.exe`, it will see the `.zip` trailer at the end of the `.exe`,
and automagically look inside to find and execute `__main__.py`. Easy-peasy.

### Why does this exist?

I probably could have used Vinay's C++ implementation from `distlib`, but what's the fun in that? In
particular, optimizing for binary size was entertaining (these are ~7x smaller than the distlib,
which doesn't matter much, but does a little bit, considering that it gets added to every Python
script). There are also some minor advantages, like I think the Rust code is easier to understand
(multiple files!) and it's convenient to be able to straightforwardly code the Python-finding logic
we want. But mostly it was just an interesting challenge.

This does owe a _lot_ to the `distlib` implementation though. The overall logic is copied
more-or-less directly.

### Anything I should know for hacking on this?

In order to minimize binary size, this uses, `panic="abort"`, and carefully avoids using
`core::fmt`. This removes a bunch of runtime overhead: by default, Rust "hello world" on Windows is
~150 KB! So these binaries are ~10x smaller.

Of course the tradeoff is that this is an awkward super-limited environment. No C runtime and
limited platform APIs... you don't even panicking support by default. To work around this:

- We use `windows` to access Win32 APIs directly. Who needs a C runtime? Though uh, this does mean
  that literally all of our code is `unsafe`. Sorry!

- `diagnostics.rs` uses `ufmt` and some cute Windows tricks to get a convenient version of
  `eprintln!` that works without `core::fmt`, and automatically prints to either the console if
  available or pops up a message box if not.

- All the meat is in `bounce.rs`.

Miscellaneous tips:

- `cargo-bloat` is a useful tool for checking what code is ending up in the final binary and how
  much space it's taking. (It makes it very obvious whether you've pulled in `core::fmt`!)

- Lots of Rust built-in panicking checks will pull in `core::fmt`, e.g., if you ever use `.unwrap()`
  then suddenly our binaries double in size, because the `if foo.is_none() { panic!(...) }` that's
  hidden inside `.unwrap()` will invoke `core::fmt`, even if the unwrap will actually never fail.
  `.unwrap_unchecked()` avoids this. Similar for `slice[idx]` vs `slice.get_unchecked(idx)`.

### How do you build this stupid thing?

Building this can be frustrating, because the low-level compiler/runtime machinery have a bunch of
implicit assumptions about the environment they'll run in, and the facilities it provides for things
like `memcpy`, unwinding, etc. So we need to replace the bits that we actually need, and which bits
we need can change depending on stuff like optimization options. For example: we use
`panic="abort"`, so we don't actually need unwinding support, but at lower optimization levels the
compiler might not realize that, and still emit references to the unwinding
helper`__CxxFrameHandler3`. And then the linker blows up because that symbol doesn't exist.

```
cargo build --release --target i686-pc-windows-msvc
cargo build --release --target x86_64-pc-windows-msvc
cargo build --release --target aarch64-pc-windows-msvc
```
