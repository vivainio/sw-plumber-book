---
icon: lucide/cog
---

# Rust and Cargo

Cargo collapses three roles that every other tool in this book keeps
separate: build system, package manager, and task runner, all driven from
one manifest and one binary. There's no equivalent of choosing Maven vs.
a separate `npm`-style installer vs. a separate task runner — `cargo
build`, `cargo add`, and `cargo test` are the same tool wearing three
hats, and the manifest that drives all three (`Cargo.toml`) is
deliberately small compared to a `pom.xml`, because most of what a
`pom.xml` spends XML on is convention Cargo just hardcodes.

## Cargo.toml and Cargo.lock: requirements versus resolution

`Cargo.toml` states what a crate needs, as a version *requirement*, not a
pinned version:

```toml
[package]
name = "my-app"
version = "0.1.0"
edition = "2021"

[dependencies]
serde = "1.0.190"
tokio = { version = "1", features = ["full"] }
```

An unadorned `"1.0.190"` is shorthand for a caret requirement,
`^1.0.190` — "any `1.x.y` with `x.y` ≥ `0.190`, but nothing that bumps the
leftmost nonzero component." This is npm's `^` behavior, not Maven's
exact-coordinate default; Cargo made semver-compatible ranges the
*default* dependency syntax rather than an opt-in. `Cargo.lock`, generated
by the first build and updated by `cargo update`, is what actually pins
every dependency (direct and transitive) to one resolved version and
checksum — the same requirements-vs-resolution split the
[Dependency Resolution](../dependency-resolution/index.md) chapter covers
in the abstract, concretely: `Cargo.toml` is what you wrote, `Cargo.lock`
is what you're actually building against, and only the lockfile is what
makes two builds on two machines reproducible. Binary crates commit
`Cargo.lock`; library crates conventionally don't, because a library's
consumers resolve its dependencies against *their own* lock, and a
library-authored lockfile would never be consulted anyway.

## The registry: crates.io and the sparse index

Resolving `serde = "1.0.190"` means asking "what versions of `serde`
exist, and what does each one depend on" without downloading every
candidate crate first. Cargo answers this from an **index** — for
crates.io, a separate git-backed (historically) or, since Cargo 1.68,
HTTP-served **sparse index**: one small JSON file per crate name, each
line a published version's metadata (dependencies, feature list,
checksum), fetched over plain HTTPS as needed rather than cloning a
multi-gigabyte index repository up front the way the old git-index
scheme required. This is the same problem Maven Central's
`maven-metadata.xml` and npm's registry API solve for their own
ecosystems, just with the index format itself being the thing that
changed shape as crates.io scaled — the git-index approach that worked
fine for a smaller registry stopped being it once fetching the whole
index became the slowest part of a fresh `cargo build`.

## Profiles: dev and release aren't a flag, they're two different builds

`cargo build` and `cargo build --release` don't just add `-O` — they
select an entire named **profile** from `Cargo.toml` (or Cargo's own
defaults if none is declared), each with its own optimization level,
debug-info setting, and codegen strategy:

```toml
[profile.release]
opt-level = 3
lto = "thin"
codegen-units = 1
panic = "abort"
```

`codegen-units = 1` forces the whole crate through a single LLVM codegen
unit instead of the default parallel split (16 for release builds),
trading build time for the optimizer being able to see and inline across
what would otherwise be unit boundaries — a real, measurable runtime win
for hot code, and a real, measurable compile-time cost, which is why it's
opt-in rather than default. `lto = "thin"` (or `"fat"`) extends that same
whole-program visibility across crate boundaries, into dependencies
themselves — the reason a release build of a dependency-heavy binary can
take minutes where the equivalent dev build takes seconds isn't the
optimizer flag alone, it's LTO reopening and re-optimizing code that dev
builds compile once and leave alone.

## Compilation units are crates, not files

Where `javac` and `tsc` compile file-by-file (with whole-program
awareness layered on top), `rustc`'s unit of compilation is the **crate**
— one invocation of `rustc` typically compiles an entire library or
binary crate's source tree in one pass, which is what makes cross-module
inlining and monomorphization work without a separate whole-program
analysis step, and also why touching one file in a large crate can
trigger recompiling the whole crate rather than just that file. Cargo's
incremental-build story operates one level below that: **incremental
compilation** (on by default for dev profiles) caches per-function
codegen results inside `target/debug/incremental/`, so `rustc` itself can
skip re-codegenning functions whose MIR didn't change, even though the
crate is still recompiled as one logical unit. This is a different shape
of incrementality than Make's per-file timestamp graph or Gradle's
per-task input hashing — it's happening *inside* a single compiler
invocation, invisible to Cargo, which only ever sees "did this crate's
fingerprint change, yes or no."

## Fingerprinting: why `target/` sometimes rebuilds "for no reason"

Cargo decides whether to recompile a crate by comparing a **fingerprint**
— a hash of the crate's source files, its `Cargo.toml`, the exact
resolved dependency versions, the enabled feature set, the target triple,
and the active profile's flags — against the fingerprint recorded the
last time it was built, stored under `target/debug/.fingerprint/`. Any
input included in that hash changing invalidates the cache; anything
*not* included (an environment variable a `build.rs` reads without
declaring it via `cargo:rerun-if-env-changed`, a file read outside
Cargo's tracked inputs) can go stale silently, which is exactly the same
failure shape the [MSBuild](../msbuild/index.md) chapter describes for
`Inputs`/`Outputs` on incremental targets — a fast, correct cache right
up until something reads state the cache doesn't know to watch.
Switching feature flags is the everyday version of this surprising
people: `cargo build --features foo` and a plain `cargo build`
fingerprint differently, so alternating between them rebuilds the
affected crates every time, not because Cargo is broken but because they
genuinely are different artifacts occupying the same `target/debug/`
directory.

## Workspaces: one lockfile, one target directory, many crates

A workspace turns a directory of related crates into one resolution unit,
declared in a root `Cargo.toml` with no `[package]` of its own:

```toml
[workspace]
members = ["core", "api", "cli"]
resolver = "2"
```

Every member crate shares a single `Cargo.lock` at the workspace root —
so `core` and `cli` can never end up depending on two different resolved
versions of the same dependency, the way two independently-locked npm
packages could — and a single `target/` build cache, so a dependency
built once for `core` is reused unmodified when `cli` also needs it, not
rebuilt per member. This is the direct analogue of Maven's reactor:
`cargo build --workspace` is `mvn install` from a parent POM, and `cargo
build -p cli` building just one member (plus what it depends on) is `mvn
install -pl cli -am`. `resolver = "2"` matters specifically for feature
unification, below — workspaces created before Cargo 1.51 default to the
older resolver unless this line is added explicitly.

## Feature unification: features are additive, and that's the trap

A Cargo **feature** gates optional code behind a `cfg` flag and,
usually, optional dependencies:

```toml
[features]
default = ["std"]
std = []
serde-support = ["dep:serde"]
```

Features are additive by design — enabling one is never allowed to
*remove* capability, only add it — which sounds harmless until a
workspace or dependency graph pulls in the same crate twice with
different feature requests: crate `A` depends on `logging` with no
extra features, crate `B` depends on `logging` with `features =
["async"]`. Cargo doesn't build two copies; it builds **one** copy of
`logging` with the *union* of every feature any dependent asked for,
because features are supposed to be additive and non-conflicting. In
practice this means a feature that isn't purely additive — one that
changes a default, or is genuinely incompatible with another feature —
silently "leaks" into a build that never explicitly asked for it, purely
because some other crate three hops away in the graph enabled it. Cargo
1.51's per-package feature resolution (`resolver = "2"`) narrowed this
for the dev-dependency/build-dependency/target-specific cases, but the
core additive-union behavior for normal dependencies is unconditional —
there's no equivalent of Maven's "nearest wins" here, because unlike
version conflicts, Cargo doesn't treat feature conflicts as something to
pick a winner for.

## build.rs: escaping the manifest into arbitrary code

Some crates need to run real logic before `rustc` ever sees the actual
source — generating code from a `.proto` file, detecting a system
library via `pkg-config`, or emitting linker flags for a vendored C
library. Cargo's answer, its equivalent of a Maven Mojo bound to an
early lifecycle phase, is a `build.rs` file at the crate root, compiled
and run as an ordinary Rust binary *before* the crate itself:

```rust
// build.rs
fn main() {
    println!("cargo:rustc-link-lib=z");
    println!("cargo:rerun-if-changed=src/native.c");
}
```

`build.rs` talks back to Cargo exclusively through `cargo:` lines printed
to stdout — `cargo:rustc-link-lib` adds a linker argument to the crate
being built, `cargo:rerun-if-changed` declares a fingerprint input Cargo
wouldn't otherwise know to watch (the exact "declare what you actually
read" discipline the fingerprinting section above depends on to stay
correct). This is also the mechanism `cc`-crate-based native bindings and
`bindgen`-based FFI wrappers build on: a `build.rs` that shells out to a
C compiler is doing, by hand, roughly what `setuptools.Extension` and
`maturin` automate for Python — except here it's the *default*,
unremarkable way a crate links against non-Rust code, not a separate
packaging path.

## Proc macros: compiled for the host, not the target

A procedural macro crate (`#[proc_macro]`, `#[derive(...)]`
implementations) runs *during* another crate's compilation, as code
loaded into `rustc` itself — which means it has to be compiled for
whatever architecture is running the compiler, never for the crate's
final target. Cross-compiling a binary for `aarch64-unknown-linux-gnu`
from an x86_64 host still builds every proc-macro dependency in that
graph as an ordinary x86_64 host binary; only the leaf binary and its
non-macro dependencies actually target `aarch64`. Cargo handles this
without being asked — it's the same "two different architectures in one
build, silently" situation Java annotation processors and MSBuild
source generators both handle by running on the host JVM/.NET runtime
regardless of what the final artifact targets — but it's also why a
crate's build can require *two* full toolchains' worth of dependency
compilation the moment cross-compilation and proc macros combine, which
shows up as unexpectedly long "resolving and compiling" phases on a
cross-compiled build that looks, from `Cargo.toml`, like it should only
need one target.

## `cargo check` versus `cargo build`: the same frontend, a different backend

`cargo check` runs the compiler's parsing, type-checking, and borrow-check
passes and stops — no codegen, no linking, no `target/debug/my-app`
binary — which makes it dramatically faster for the "does this even
compile" loop than a full `cargo build`, and is what most editor
integrations (`rust-analyzer`) run on every keystroke rather than a real
build. The two share a compilation cache where the type-checking work
overlaps, so running `cargo check` first and then `cargo build` doesn't
redo the checking phase from scratch — but the reverse assumption, that
`cargo check` passing means `cargo build` will succeed, holds for
ordinary code and stops holding the moment a `build.rs` or proc macro has
its own runtime failure mode that only a real compile-and-run exercises.

## Where people actually get burned

- **`Cargo.lock` committed for a library.** The convention is
  library crates omit it from version control (consumers resolve
  against their own lock) while binary/application crates commit it.
  Doing the reverse for a library either bakes in stale transitive pins
  that never get exercised, or — worse — gets committed, ignored by every
  downstream consumer's own resolution, and quietly rots.
- **Feature unification pulling in something a crate never asked for.**
  A `no_std` crate that ends up linking `std` anyway because a sibling
  dependency three hops away enabled a `std` feature on a shared
  dependency is almost always this, not a bug in the crate's own
  `Cargo.toml` — `cargo tree -e features` is the direct way to see which
  dependency actually requested the feature that leaked.
- **Switching `--features` flags thrashing the build cache.** Alternating
  between `cargo build` and `cargo build --all-features` locally rebuilds
  large parts of the dependency graph each time, because the feature set
  is part of the fingerprint — this is expected, not a caching bug, and
  the usual fix is two separate `target-dir`s (`CARGO_TARGET_DIR`) rather
  than fighting the fingerprint.
- **A `build.rs` that reads state it never declares.** An environment
  variable read without `cargo:rerun-if-env-changed`, or a file opened
  without `cargo:rerun-if-changed`, means Cargo has no way to know the
  build script's output is stale — the build "just doesn't pick up" a
  changed value until something else invalidates the fingerprint by
  accident, which reads as a flaky build script when it's actually an
  under-declared one.
