---
icon: lucide/box-select
---

# Bazel

Every build tool earlier in this book answers "what needs to rerun?" by
comparing timestamps or, at best, hashing a few known inputs. Bazel
(Google's internal "Blaze," open-sourced in 2015) answers a stricter
question: given the *exact* set of inputs a step depends on — every file,
every environment variable, every tool version, and nothing else — has
that exact set changed? Getting that question right requires refusing to
let a build step see anything it didn't declare, which is a fundamentally
different posture than Make (any recipe can shell out and read whatever
it wants) or Gradle (dependencies are declared, but a task body is
arbitrary Groovy/Kotlin that can still reach outside its declared inputs).
Bazel calls the property **hermeticity**, and almost every unusual thing
about it — the sandboxing, the Starlark language restrictions, the
insistence on declaring every dependency explicitly — exists in service
of that one property.

## Actions, not tasks

Gradle and MSBuild build a graph of *tasks*: named, user-visible units of
work with imperative bodies. Bazel builds a graph of **actions**: a
command line, an explicit list of input files, and an explicit list of
output files, with no other implicit inputs allowed. A `cc_library` rule
doesn't run "the compile task" — it generates one action per source file,
each one exactly `gcc -c foo.c -o foo.o` (roughly) plus the precise set of
headers, flags, and toolchain files that action is allowed to see.

The **action key** is a hash of the command line and every declared
input's content — not its timestamp, its actual hash. If that key has
been computed before, on this machine or a remote cache, Bazel skips
running the action entirely and fetches the output instead. This is why
Bazel remote caching works across completely different machines with no
shared filesystem or clock: cache lookup is content-addressed, so two
unrelated machines building identical inputs land on the same key and one
can serve the other's result.

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

Every target has a **label** — `//src/mylib:mylib` — that's globally
unique across the whole workspace, unlike a Makefile's targets, which
only mean something relative to the makefile that defined them. `deps =
["//src/utils:utils"]` is not a suggestion or a classpath entry; it's the
entire universe of what `mylib.c`'s compile action is allowed to read
besides its own declared `srcs`/`hdrs`. Miss a header out of `hdrs` and
the build doesn't quietly work anyway the way it would with Make — it
fails, because the sandbox (next) never made that header visible in the
first place.

## Sandboxing: enforcing hermeticity, not just hoping for it

Declaring inputs is worthless if a compiler can still `#include` a header
that happens to sit in the same directory but was never declared. Bazel
enforces the declaration by running each action inside a **sandbox**: a
symlink forest containing *only* the declared inputs, nothing else from
the real source tree. If `mylib.c` `#include`s a header nobody listed in
`hdrs`/`deps`, the compile fails with a file-not-found error even though
the header exists two directories up in the real checkout — because the
sandboxed view of the filesystem never included it. This is the single
biggest source of friction migrating an existing C++ or Java project to
Bazel: undeclared dependencies that "worked" for years under Make or
Maven, because the compiler's working directory happened to make them
visible, all surface at once as build failures. It's also exactly the
point — those were latent bugs (a build that only works because of
directory layout accidents) that Bazel refuses to let pass silently.

## Starlark: a deliberately restricted language

`BUILD` files and `.bzl` macro/rule definitions are written in
**Starlark**, a dialect of Python with entire features removed: no
classes, no unbounded loops without a fixed iteration limit, no I/O, no
`import` of arbitrary modules, no mutable global state visible across
evaluations. None of this is arbitrary austerity — every removed feature
is a way a build file could otherwise produce different output on
different machines or different runs (reading the current time, hitting
the network, depending on dict iteration order in old Python). A language
that can't do those things is a language whose evaluation is safe to
cache, safe to run in parallel, and safe to distribute — the same
hermeticity goal, applied to the build *description* instead of just the
build *actions*.

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

External dependencies (other repositories, prebuilt archives, other
package ecosystems) were historically declared in a `WORKSPACE` file with
manually-specified download URLs and content hashes — hermetic in the
same sense as everything else (a pinned hash, not a floating version) but
notoriously bad at resolving version conflicts between transitive
dependencies, since `WORKSPACE` had no real resolution algorithm, just
first-one-wins. **Bzlmod** (`MODULE.bazel`), Bazel's newer dependency
system, replaces this with real version resolution (minimal version
selection, similar in spirit to Go modules) while keeping the same
underlying hermeticity guarantee — every resolved dependency still
bottoms out in a content hash, just arrived at through an actual
algorithm instead of file-declaration order.

## Introspecting the graph: `bazel query`

Because the entire dependency graph is explicit — no step can have a
dependency it didn't declare — it's also fully queryable, not just
executable:

```sh
bazel query 'deps(//src/mylib:app)'
bazel query 'rdeps(//..., //src/utils:utils)'   # what depends on utils?
bazel query 'somepath(//src/mylib:app, //src/legacy:old_lib)'
```

`rdeps` (reverse dependencies) answers "what breaks if I change this
target?" directly from the graph, before running any build at all — a
question Make or Gradle can only answer by grepping build files by hand,
since neither maintains the graph as a first-class, query-able structure
independent of actually executing it.

## Why the friction is usually the point

Bazel has a reputation for being heavyweight to adopt, and the reputation
is earned: every external dependency needs an explicit, hashed
declaration, every target needs accurate `deps`, and there's no
"reasonable default" escape hatch the way Make lets a recipe just shell
out to whatever's on `$PATH`. That friction is the hermeticity property
showing up at build-file-authoring time instead of at "why did this build
break on a different machine" time — the same correctness Make and
Gradle chapters described as their sharp edges (timestamps as a leaky
proxy, task bodies with unchecked side effects) is what Bazel spends its
extra ceremony to close off entirely, at the cost of never being the
quick, no-config option for a small project that a `Makefile` still is.
