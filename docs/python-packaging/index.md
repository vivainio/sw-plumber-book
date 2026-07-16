---
icon: lucide/package-2
---

# Python Packaging: Eggs, Wheels, and Native Code

Python's packaging story is messier than Maven's or npm's for a structural
reason: a "Python package" is sometimes pure Python and sometimes a thin
wrapper around compiled C, Rust, or Fortran, and the two cases need
completely different build steps to turn into something `pip install`able.
JARs are always bytecode; npm packages are almost always JavaScript. A wheel
might be a zip full of `.py` files, or it might contain a `.so` compiled
against a specific CPython ABI, a specific OS, and a specific CPU
architecture — and the filename has to say which, because there's no JVM or
V8 underneath smoothing the difference away.

## sdist: the source distribution, and why it isn't enough on its own

`python -m build --sdist` produces a `.tar.gz` containing your source tree
plus generated metadata — conceptually the equivalent of `mvn source:jar`,
except it's usually the *primary* artifact `setup.py sdist` produced for
decades, not a side one. Installing directly from an sdist means running
its build step on the *target* machine at install time: fine for pure
Python, but for anything with a C extension it means the installing
machine needs a working C compiler, the right header files, and often the
exact development libraries the extension links against (`libpq-dev` for
`psycopg2`, `libxml2-dev` for `lxml`). This was the normal, expected
failure mode of `pip install` for most of Python's history — a red wall of
compiler errors on a machine that just didn't have `gcc` installed — and
it's the entire reason the wheel format exists.

## Eggs: setuptools' first attempt, and why it didn't last

Before wheels, `setuptools` shipped **eggs** (`.egg` files) alongside
`easy_install`, installed via `python setup.py bdist_egg`. An egg is a zip
(or, in "unzipped" mode, a bare directory) with the built package plus an
`EGG-INFO/` metadata directory — structurally similar to what a wheel does,
but eggs could contain compiled code *and* be directly importable at
runtime by adding themselves to `sys.path`, which meant `easy_install`
supported installing a package as a live zip that Python imported from
without ever fully unpacking it. That flexibility turned out to be the
problem: eggs had no standardized, tool-independent metadata format (their
`EGG-INFO` was setuptools-specific, not a PEP-defined spec), no reliable
way to express "this is compatible with these platforms" beyond a coarse
tag, and the zip-import mode broke tools that assumed a package's files
existed as real files on disk — the same class of problem Yarn's PnP would
later hit for the same reason, just a decade earlier and for a different
ecosystem. **PEP 427** defined the wheel format specifically to fix this:
a standardized, install-tool-agnostic layout with metadata any tool could
read, and no live-zip-import ambiguity. `easy_install` and `.egg` are
legacy at this point — no current tooling produces them by default, and
seeing one in the wild today almost always means a very old package that
hasn't been rebuilt since.

## Wheels: what's actually inside one

A wheel is just a zip file with a `.whl` extension and a rigid internal
layout:

```
$ unzip -l requests-2.31.0-py3-none-any.whl
requests/__init__.py
requests/api.py
requests/models.py
...
requests-2.31.0.dist-info/METADATA
requests-2.31.0.dist-info/RECORD
requests-2.31.0.dist-info/WHEEL
requests-2.31.0.dist-info/entry_points.txt
```

`METADATA` holds the package name, version, and dependency list (the
`Requires-Dist` lines `pip` reads to resolve the dependency graph);
`RECORD` lists every file in the wheel with a hash and size, which is what
lets `pip uninstall` remove exactly the files it installed and nothing
else; `WHEEL` records the tool and wheel-spec version that built it.
Installing a wheel is, deliberately, almost mechanical: unzip it into
`site-packages`, write the `RECORD`-listed files, done — no build step,
no `setup.py` execution, which is precisely what makes wheels fast and
what egg's live-import trick never reliably offered.

## The filename is the compatibility contract

```
requests-2.31.0-py3-none-any.whl
cryptography-42.0.5-cp311-cp311-manylinux_2_28_x86_64.whl
```

The pattern is `{name}-{version}-{python tag}-{abi tag}-{platform tag}.whl`.
`py3-none-any` means "any Python 3 interpreter, no ABI constraint, any
platform" — pure Python, installable anywhere. `cp311-cp311-manylinux_2_28_x86_64`
means "built against CPython 3.11's specific C ABI, for glibc-based Linux
on x86_64, compatible with distros new enough to satisfy `manylinux_2_28`."
`pip` doesn't guess compatibility — it parses this filename against the
running interpreter and platform, and if nothing on the index matches, it
falls back to the sdist and tries to build from source right there, which
is often the moment a `pip install` that "should just work" suddenly needs
a compiler after all.

## manylinux: solving "compiled where" for a platform with no ABI stability

A wheel with compiled code built on Ubuntu 22.04 isn't automatically safe
to run on Ubuntu 18.04, RHEL 7, or Alpine — Linux offers no stable
userspace C ABI the way Windows or macOS effectively do, and a `.so` linked
against a newer glibc can fail to even load (`GLIBC_2.34 not found`) on an
older system. **manylinux** is PyPI's answer: a set of PEP-defined
platform tags (`manylinux_2_17`, `manylinux_2_28`, and the versionless
predecessors `manylinux1`/`manylinux2010`/`manylinux2014`), each backed by
a published Docker image built against an old-enough glibc/toolchain that
a wheel compiled inside it will run correctly on essentially every
mainstream distro released since. Publishing a manylinux wheel means
building inside that container, then running **`auditwheel repair`** on
the output, which inspects the compiled extension's dynamic symbol table,
verifies every glibc symbol it references is old enough for the claimed
tag, and — critically — **vendors** any non-standard shared library
dependencies (bundling `libssl.so` into the wheel itself, with its symbols
renamed to avoid colliding with the host's own copy) so the wheel doesn't
silently depend on a system library that may or may not be present at
install time. Skipping this step and uploading a wheel built on your own
laptop is the single most common way to ship a package that installs fine
for you and segfaults or refuses to import for a meaningful slice of your
users.

## pyproject.toml and build backends: PEP 517/518

Modern builds are backend-agnostic. `pyproject.toml` declares which tool
turns source into a wheel:

```toml
[build-system]
requires = ["setuptools>=68", "wheel"]
build-backend = "setuptools.build_meta"
```

`python -m build` reads this, creates an **isolated virtual environment**
containing exactly the `requires` packages (never the ambient environment,
by design — this is PEP 517/518's actual point), and calls the backend's
`build_wheel()`/`build_sdist()` hooks inside it. `setuptools.build_meta` is
one backend among several — `hatchling`, `flit_core`, `pdm-backend`, and
`maturin` (below) are others — and the reason this matters is that
`setup.py` stopped being *the* build step and became just one backend's
implementation detail; a package built with a `hatchling` backend has no
`setup.py` at all, and running `python setup.py install` against it simply
fails because there's no such file to run.

## Native extensions the classical way: setuptools and `Extension`

A C extension declared for `setuptools.build_meta` names its sources
directly:

```python
# setup.py
from setuptools import setup, Extension

setup(
    ext_modules=[
        Extension("mypkg._native", sources=["mypkg/_native.c"]),
    ],
)
```

`python -m build` invokes the platform's C compiler (`cc`/`gcc`/`clang` on
Unix, `cl.exe` on Windows via the same MSVC toolchain the
[MSBuild](../msbuild/index.md) chapter's `Csc` task lives next to) against
Python's own `Python.h` headers, and the compiled artifact lands inside
the wheel with an ABI-tagged filename of its own:

```
mypkg/_native.cpython-311-x86_64-linux-gnu.so
```

`import mypkg._native` works because CPython's import system, when
searching for extension modules, looks specifically for a filename
carrying its own `cpython-311-...` tag (or, for the ABI-stable subset
below, `abi3`) — an extension built for 3.10 does not satisfy an import
under 3.11, full stop, no fallback, which is exactly why a project
publishing compiled wheels has to publish one per supported Python
minor version.

## The stable ABI: `abi3`, and why it's worth targeting

CPython's C API changes in incompatible ways between minor versions by
default, which is why extensions are normally rebuilt per version. A
deliberately frozen subset of that API — the **limited API** — is
guaranteed stable across CPython 3.x releases, and an extension built
against only that subset can declare itself `Py_LIMITED_API`, producing a
single `abi3` wheel (`mypkg-1.0.0-cp38-abi3-manylinux_2_28_x86_64.whl`)
that installs correctly on 3.8 *and every later 3.x*, instead of one wheel
per minor version. The tradeoff is real: the limited API is a strict
subset, so anything using newer, faster, or more ergonomic C API additions
introduced after the targeted floor version isn't available — projects
choose `abi3` for release-matrix simplicity, and opt back into
version-specific builds when they need something outside that frozen
surface.

## maturin and PyO3: Rust extensions without hand-written C glue

Writing a CPython C extension by hand means manually managing reference
counts, GIL state, and exception propagation across the C boundary — a
well-known source of use-after-free and refcount bugs even in careful
code. **PyO3** is a Rust crate that wraps CPython's C API behind safe Rust
types and procedural macros:

```rust
use pyo3::prelude::*;

#[pyfunction]
fn fast_sum(values: Vec<i64>) -> i64 {
    values.iter().sum()
}

#[pymodule]
fn _native(m: &Bound<'_, PyModule>) -> PyResult<()> {
    m.add_function(wrap_pyfunction!(fast_sum, m)?)?;
    Ok(())
}
```

**maturin** is the build backend that turns this into a wheel — it's a
`pyproject.toml` `build-backend` entry exactly like `setuptools.build_meta`
or `hatchling`, just backed by `cargo` instead of a C compiler:

```toml
[build-system]
requires = ["maturin>=1.0,<2.0"]
build-backend = "maturin"
```

`maturin develop` builds the Rust crate and installs it directly into the
active virtualenv for local iteration — the Rust-toolchain equivalent of
`pip install -e .` — while `maturin build --release` produces the
actual manylinux-tagged wheel for distribution, running the moral
equivalent of `auditwheel repair` internally (`maturin` links against a
pinned-old-enough glibc target itself, via `zig cc` or a manylinux
container, rather than requiring a separate repair pass). PyO3 defaults
new projects to `abi3` support out of the box, which combined with Rust's
memory safety guarantees is the specific combination — no per-version
rebuild matrix, no manual refcounting — that's made Rust the default
choice for new performance-critical Python extensions over hand-written C,
independent of Rust's other language merits.

## cibuildwheel: the manylinux/macOS/Windows matrix problem

A package with compiled code that wants to support Linux, macOS
(x86_64 and arm64), and Windows, across several Python versions, needs a
different build environment for essentially every cell in that matrix —
nobody's laptop can natively produce a manylinux-compliant Linux wheel
*and* a codesigned-compatible macOS wheel *and* an MSVC-built Windows
wheel. **cibuildwheel** is a CI-oriented tool, typically run as a GitHub
Actions step, that drives the correct build environment per target (the
manylinux Docker images on Linux, the right Xcode SDK on macOS, MSVC on
Windows) and runs `auditwheel`/`delocate`/`delvewheel` (the macOS and
Windows equivalents) automatically, producing the full wheel matrix as CI
artifacts in one pipeline run — the compiled-extension version of the
[CI/CD pipelines](../cicd-pipelines/index.md) chapter's point that
orchestration and the build itself are different layers, here applied to
a build matrix no single machine can produce alone.

## Where people actually get burned

- **A wheel built on your dev machine "for Linux" isn't a portable Linux
  wheel.** `pip wheel .` on a bare Ubuntu box produces a `linux_x86_64`
  tagged wheel, which PyPI **rejects on upload** precisely because it
  carries no manylinux compatibility guarantee — only `auditwheel repair`
  (or a proper manylinux container build) produces something PyPI accepts
  and other machines can trust.
- **`pip install --no-binary` forcing a source build.** Passing
  `--no-binary :all:` (or a package hitting it implicitly because no
  matching wheel tag exists for an unusual platform) silently falls back
  to sdist compilation, and the resulting "missing `gcc`" or "missing
  Python.h" error reads like a broken package when it's actually a missing
  compatible wheel.
- **Mixing ABI-tagged and `abi3` extensions in one project.** A project
  with some modules built `abi3` and others built per-version can end up
  shipping a wheel whose filename claims broad compatibility
  (`cp38-abi3-...`) while one of its own `.so` files was accidentally
  compiled without `Py_LIMITED_API` — it imports fine under the Python
  version it was actually built with and throws a cryptic
  `undefined symbol` or version-mismatch error under any other, because
  the outer wheel tag and the inner extension's real ABI disagreed.
- **A vendored shared library colliding with the system's own copy.**
  `auditwheel repair`'s symbol-renaming step exists specifically to avoid
  this, but hand-rolled repair processes (or older tooling) that skip it
  can produce a wheel whose bundled `libcrypto.so` silently shadows or
  conflicts with whatever OpenSSL the rest of the process already loaded
  — a bug that only appears when two wheels vendoring different versions
  of the same underlying library end up imported in the same process.
