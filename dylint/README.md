# Dylint

A tool for running Rust lints from dynamic libraries

```sh
cargo install cargo-dylint dylint-link
```

Dylint is a Rust linting tool, similar to Clippy. But whereas Clippy runs a predetermined, static set of lints, Dylint runs lints from user-specified, dynamic libraries. Thus, Dylint allows developers to maintain their own personal lint collections.

**Contents**

- [Quick start]
  - [Running Dylint]
  - [Writing lints]
- [Features]
  - [Workspace metadata]
  - [Configurable libraries]
  - [Conditional compilation]
  - [VS Code integration]
- [Utilities]
- [Resources]

Documentation is also available on [how Dylint works].

## Quick start

### Running Dylint

The next three steps install Dylint and run all of this repository's [general-purpose, example lints] on a workspace:

1. Install `cargo-dylint` and `dylint-link`:

   ```sh
   cargo install cargo-dylint dylint-link
   ```

2. Add the following to the workspace's `Cargo.toml` file:

   ```toml
   [workspace.metadata.dylint]
   libraries = [
       { git = "https://github.com/trailofbits/dylint", pattern = "examples/general/*" },
   ]
   ```

3. Run `cargo-dylint`:
   ```sh
   cargo dylint --all --workspace
   ```

In the above example, the libraries are found via [workspace metadata], which is the recommended way. For additional ways of finding libraries, see [How Dylint works].

### Writing lints

You can start writing your own Dylint library by running `cargo dylint new new_lint_name`. Doing so will produce a loadable library right out of the box. You can verify this as follows:

```sh
cargo dylint new new_lint_name
cd new_lint_name
cargo build
DYLINT_LIBRARY_PATH=$PWD/target/debug cargo dylint list --lib new_lint_name
```

All you have to do is implement the [`LateLintPass`] trait and accommodate the symbols asking to be filled in.

Helpful [resources] for writing lints appear below.

## Features

### Workspace metadata

A workspace can name the libraries it should be linted with in its `Cargo.toml` file. Specifically, a workspace's manifest can contain a TOML list under `workspace.metadata.dylint.libraries`. Each list entry must have the form of a Cargo `git` or `path` dependency, with the following differences:

- There is no leading package name, i.e., no `package =`.
- `path` entries can contain [glob] patterns, e.g., `*`.
- Any entry can contain a `pattern` field whose value is a [glob] pattern. The `pattern` field indicates the subdirectories that contain Dylint libraries.

Dylint downloads and builds each entry, similar to how Cargo downloads and builds a dependency. The resulting `target/release` directories are searched for files with names of the form that Dylint recognizes (see [Library requirements] under [How Dylint works]).

As an example, if you include the following in your workspace's `Cargo.toml` file and run `cargo dylint --all --workspace`, Dylint will run on your workspace all of this repository's [example general-purpose lints], as well as the example restriction lint [`try_io_result`].

```toml
[workspace.metadata.dylint]
libraries = [
    { git = "https://github.com/trailofbits/dylint", pattern = "examples/general/*" },
    { git = "https://github.com/trailofbits/dylint", pattern = "examples/restriction/try_io_result" },
]
```

### Configurable libraries

Libraries can be configured by including a `dylint.toml` file in a linted workspace's root directory. The file should encode a [toml table] whose keys are library names. A library determines how its value in the table (if any) is interpreted.

As an example, a `dylint.toml` file with the following contents sets the [`non_local_effect_before_error_return`] library's `work_limit` configuration to `1_000_000`:

```toml
[non_local_effect_before_error_return]
work_limit = 1_000_000
```

For instructions on creating a configurable library, see the [`dylint_linting`] documentation.

### Conditional compilation

For each library that Dylint uses to check a crate, Dylint passes the following to the Rust compiler:

```sh
--cfg=dylint_lib="LIBRARY_NAME"
```

You can use this feature to allow a lint when Dylint is used, but also avoid an "unknown lint" warning when Dylint is not used. Specifically, you can do the following:

```rust
#[cfg_attr(dylint_lib = "LIBRARY_NAME", allow(LINT_NAME))]
```

Note that `LIBRARY_NAME` and `LINT_NAME` may be the same. For an example involving [`non_thread_safe_call_in_test`], see [dylint/src/lib.rs] in this repository.

Also note that the just described approach does not work for pre-expansion lints. The only known workaround for pre-expansion lints is allow the compiler's built-in [`unknown_lints`] lint. Specifically, you can do the following:

```rust
#[allow(unknown_lints)]
#[allow(PRE_EXPANSION_LINT_NAME)]
```

For an example involving [`env_cargo_path`], see [internal/src/examples.rs] in this repository.

### VS Code integration

Dylint results can be viewed in VS Code using [rust-analyzer]. To do so, add the following to your VS Code `settings.json` file:

```json
    "rust-analyzer.checkOnSave.overrideCommand": [
        "cargo",
        "dylint",
        "--all",
        "--workspace",
        "--",
        "--all-targets",
        "--message-format=json"
    ]
```

If you want to use rust-analyzer inside a lint library, you need to add the following to your VS Code `settings.json` file:

```json
    "rust-analyzer.rustc.source": "discover",
```

And add the following to the library's `Cargo.toml` file:

```toml
[package.metadata.rust-analyzer]
rustc_private = true
```

## Utilities

The following utilities can be helpful for writing Dylint libraries:

- [`dylint-link`] is a wrapper around Rust's default linker (`cc`) that creates a copy of your library with a filename that Dylint recognizes.
- [`dylint_library!`] is a macro that automatically defines the `dylint_version` function and adds the `extern crate rustc_driver` declaration.
- [`ui_test`] is a function that can be used to test Dylint libraries. It provides convenient access to the [`compiletest_rs`] package.
- [`clippy_utils`] is a collection of utilities to make writing lints easier. It is generously made public by the Rust Clippy Developers. Note that, like `rustc`, `clippy_utils` provides no stability guarantees for its APIs.

## Resources

Helpful resources for writing lints include the following:

- [Adding a new lint] (targeted at Clippy but still useful)
- [Author lint]
- [Common tools for writing lints]
- [Guide to Rustc Development]
- [Crate `rustc_hir`]
- [Crate `rustc_middle`]
- [Struct `rustc_lint::LateContext`]
  - [Method `typeck_results`]
  - [Field `tcx`]
    - [Method `hir`]

[`clippy_utils`]: https://github.com/rust-lang/rust-clippy/tree/master/clippy_utils
[`compiletest_rs`]: https://github.com/Manishearth/compiletest-rs
[`dylint-link`]: ../dylint-link
[`dylint_library!`]: ../utils/linting
[`dylint_linting`]: ../utils/linting
[`env_cargo_path`]: ../examples/general/env_cargo_path
[`latelintpass`]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_lint/trait.LateLintPass.html
[`non_local_effect_before_error_return`]: ../examples/general/non_local_effect_before_error_return
[`non_thread_safe_call_in_test`]: ../examples/general/non_thread_safe_call_in_test
[`try_io_result`]: ../examples/restriction/try_io_result
[`ui_test`]: ../utils/testing
[`unknown_lints`]: https://doc.rust-lang.org/rustc/lints/listing/warn-by-default.html#unknown-lints
[adding a new lint]: https://github.com/rust-lang/rust-clippy/blob/master/book/src/development/adding_lints.md
[author lint]: https://github.com/rust-lang/rust-clippy/blob/master/book/src/development/adding_lints.md#author-lint
[common tools for writing lints]: https://github.com/rust-lang/rust-clippy/blob/master/book/src/development/common_tools_writing_lints.md
[conditional compilation]: #conditional-compilation
[configurable libraries]: #configurable-libraries
[crate `rustc_hir`]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_hir/index.html
[crate `rustc_middle`]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_middle/index.html
[dylint/src/lib.rs]: ../dylint/src/lib.rs
[example general-purpose lints]: ../examples/general
[features]: #features
[field `tcx`]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_lint/struct.LateContext.html#structfield.tcx
[general-purpose, example lints]: ../examples/README.md#general
[glob]: https://docs.rs/glob/0.3.0/glob/struct.Pattern.html
[guide to rustc development]: https://rustc-dev-guide.rust-lang.org/
[how dylint works]: ../docs/how_dylint_works.md
[internal/src/examples.rs]: ../internal/src/examples.rs
[library requirements]: ../docs/how_dylint_works.md#library-requirements
[method `hir`]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_middle/ty/context/struct.TyCtxt.html#method.hir
[method `typeck_results`]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_lint/struct.LateContext.html#method.typeck_results
[quick start]: #quick-start
[resources]: #resources
[running dylint]: #running-dylint
[rust-analyzer]: https://github.com/rust-analyzer/rust-analyzer
[struct `rustc_lint::latecontext`]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_lint/struct.LateContext.html
[toml table]: https://toml.io/en/v1.0.0#table
[utilities]: #utilities
[vs code integration]: #vs-code-integration
[workspace metadata]: #workspace-metadata
[writing lints]: #writing-lints
