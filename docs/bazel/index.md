---
icon: lucide/box-select
---

# Bazel

The build tools earlier in this book answer "what needs to rerun?" using
timestamps or hashes of declared inputs. Bazel
(Google's internal "Blaze," open-sourced in 2015) answers a stricter
question: given the declared inputs and execution environment of a step,
has anything relevant changed? Getting that question right means trying
to prevent a build step from observing undeclared state, which is a
fundamentally different posture than Make (a recipe can shell out and read
whatever it wants) or Gradle (a task body is arbitrary Groovy/Kotlin and
can reach outside its declared inputs). Bazel calls the desired property
**hermeticity**, and almost every unusual thing about it — sandboxing,
restricted build-definition languages, explicit dependencies, and
registered toolchains — exists in service of that property. Bazel makes
hermetic builds possible and strongly encourages them; a poorly written
rule or action can still introduce undeclared inputs.

## Actions, not tasks

Gradle and MSBuild expose graphs of *tasks*: named, user-visible units of
work that may have imperative bodies. Bazel rules analyze into
**actions**: commands with declared inputs and outputs. A `cc_library`
doesn't run one monolithic "compile task" — it normally generates a
compile action for each source file, plus linking or archiving actions as
needed. Those actions include compiler arguments, headers, toolchain
files, environment variables selected for the action, and other
rule-specific inputs.

Bazel's cache identity incorporates the action definition and the
contents of its inputs, rather than relying on file timestamps. The exact
key includes details such as arguments, selected environment variables,
execution properties, and input digests; thinking of it as "the hash of
everything relevant to this action" is more useful than memorizing an
implementation-specific formula. If an identical result exists locally
or in a remote cache, Bazel can skip execution and reuse it. This is why
remote caching works across machines with no shared filesystem or clock:
two machines presenting the same action and input contents arrive at the
same cache identity.

## BUILD files, targets, and labels

A `BUILD` (or `BUILD.bazel`) file in each directory declares **targets**
using **rules** — `cc_library`, `java_binary`, `go_test`, and hundreds of
others, either built in or defined by anyone in Starlark (below):

```python
# //src/mylib/BUILD
cc_library(
    name = "mylib",
    srcs = ["mylib.c"],
    hdrs = ["mylib.h"],
    deps = ["//src/utils:utils"],
)

cc_binary(
    name = "app",
    srcs = ["main.c"],
    deps = [":mylib"],
)
```

Every target has a **label** — `//src/mylib:mylib` — that's unique within
its repository, unlike a Makefile target whose meaning is tied to the
makefile that defined it. The `//` starts at the repository root,
`src/mylib` is the package, and the name after `:` is the target. Inside
the same package, `:mylib` is shorthand for the full label.

`deps = ["//src/utils:utils"]` declares a target-level dependency, not an
arbitrary classpath or include-path hint. Along with `srcs`, `hdrs`,
generated inputs, the selected toolchain, and rule-specific implicit
dependencies, it determines the inputs of the resulting actions. If code
uses a dependency that the rule did not declare, a correctly implemented
rule and sandboxed build should expose the mistake rather than succeeding
because of an accidental filesystem or classpath layout.

## Sandboxing: enforcing hermeticity, not just hoping for it

Declaring inputs is of limited value if a compiler can still read a header
that was never declared. Bazel therefore offers **sandboxed execution
strategies**. Depending on the operating system and selected strategy, an
action runs in an isolated working directory populated with its known
inputs, with additional operating-system restrictions where available.
This is stricter than merely running a command in the checkout.

Sandboxing is not one identical mechanism on every platform, nor does it
magically repair an incorrectly implemented rule. Some tools also use
input discovery, such as discovering included C/C++ headers, before the
final action inputs are known. The practical goal remains the same:
undeclared dependencies should fail locally instead of surviving until a
remote build or a different checkout happens not to contain them.
Surfacing those dependencies is a major source of friction when migrating
an existing project—and one of the main benefits.

## Starlark: a deliberately restricted language

`BUILD` files and `.bzl` macro/rule definitions are written in
**Starlark**, a language with Python-like syntax but deliberately fewer
ways to observe or mutate the outside world. There is no arbitrary module
import, file I/O, network access, `while` loop, or recursion. `.bzl` files
can define functions and iterate with `for`; the more declarative
`BUILD`-file dialect disallows `def`, `for`, and statement-form `if`
(comprehensions and conditional expressions are still available).
Module-level values are frozen after loading, preventing mutable global
state from leaking between evaluations.

These restrictions make loading and analysis deterministic enough to
cache and parallelize. Starlark constrains the build *description*;
sandboxing and carefully designed rules constrain the actions that
description creates.

## Remote execution: sandboxing pays off twice

Because an action's exact inputs are already fully enumerated for the
sandbox, that same enumeration is sufficient to ship the action to a
different machine entirely: send the input files (or their content
hashes, if the remote worker already has them cached), the command line,
and get back the declared outputs. This is **remote execution** —
distributing a build across a cluster of workers, not just caching
results — and it's a comparatively small step once sandboxing already
exists, versus being a bolt-on feature the way "distributed builds" are
for tools built around implicit inputs. A CI cluster building the same
monorepo from many commits benefits doubly: the remote *cache* serves
identical actions instantly, and the remote *executors* parallelize the
rest across machines instead of one CI runner's core count.

## Dependency management: WORKSPACE, and its replacement

External repositories were historically created from a `WORKSPACE` file
using repository rules such as `http_archive`. `WORKSPACE` evaluation is
ordered and repository names must be coordinated globally; it was not a
general module system with a standard transitive version-resolution
algorithm. Reusable rule sets consequently grew their own conventions
for declaring repositories and avoiding name or version conflicts.

**Bzlmod**, configured in `MODULE.bazel`, is the current module system and
the replacement for the legacy `WORKSPACE` mechanism. It resolves a
module graph using **minimal version selection**: despite the name, when
several versions of a module at the same compatibility level are
requested, the highest requested version normally wins. Registries and
lockfiles make resolution reproducible, while archive integrity hashes
can verify downloaded content. Not every repository is literally a
hashed download—local path overrides and module extensions exist—so
hermeticity still depends on how a dependency is obtained.

## A small working repository

The quickest way to turn those concepts into muscle memory is to build a
tiny project. Pin Bazel itself for the repository—usually with Bazelisk
and a checked-in `.bazelversion`—then create a module:

```python
# MODULE.bazel
module(name = "hello")
```

Add a source file and target:

```python
# hello/BUILD.bazel
cc_binary(
    name = "hello",
    srcs = ["hello.cc"],
)
```

```cpp
// hello/hello.cc
#include <iostream>

int main() {
    std::cout << "hello\n";
}
```

The core commands operate on labels:

```sh
bazel build //hello:hello     # build one target
bazel run //hello:hello       # build it, then run it
bazel test //...              # test every package below the repository root
```

`//hello:hello` is the target; `//hello` is commonly accepted as shorthand
when the target name matches the final package segment; `//...` is a
recursive target pattern. Build outputs appear through convenience
symlinks such as `bazel-bin`, backed by Bazel's output tree. Treat those
paths as outputs, not as source directories to edit.

When a build surprises you, inspect progressively deeper layers:

```sh
bazel query 'deps(//hello:hello)'   # unconfigured target graph
bazel cquery //hello:hello          # configured targets after select/toolchains
bazel aquery //hello:hello          # generated actions, inputs, and arguments
bazel clean                         # discard outputs; rarely the first remedy
```

`query` answers structural questions quickly. `cquery` includes the
configuration chosen for a particular build. `aquery` reaches the action
level and is the useful one when asking “what command will run?” or “why
is this file an input?” A normal cache or dependency problem should be
diagnosed before reaching for `clean`; deleting all outputs removes the
evidence as well as the cache.

## Introspecting the graph: `bazel query`

Because Bazel represents the declared dependency graph explicitly, it is
queryable independently of executing a build:

```sh
bazel query 'deps(//src/mylib:app)'
bazel query 'rdeps(//..., //src/utils:utils)'   # what depends on utils?
bazel query 'somepath(//src/mylib:app, //src/legacy:old_lib)'
```

`rdeps` (reverse dependencies) answers "what declares a dependency on
this target?" before running a build. That is not exactly the same as
"what will break?"—behavioral effects can escape the declared graph—but
it is a powerful approximation. Other build tools expose pieces of their
task or dependency graphs too; Bazel's advantage is a uniform query
language over a repository-wide target graph, plus `cquery` and `aquery`
for its configured and action-level forms.

## Why the friction is usually the point

Bazel has a reputation for being heavyweight to adopt, and the reputation
is earned: external dependencies and toolchains need reproducible
definitions, targets need accurate `deps`, and arbitrary shell commands
must be wrapped in actions whose inputs and outputs Bazel understands.
That friction is the hermeticity property
showing up at build-file-authoring time instead of at "why did this build
break on a different machine" time — the same correctness Make and
Gradle chapters described as their sharp edges (timestamps as a leaky
proxy, task bodies with unchecked side effects) is what Bazel spends its
extra ceremony to constrain. The payoff is strongest in large,
multi-language repositories and shared CI; for a small project, a
`Makefile` may remain the clearer and cheaper choice.
