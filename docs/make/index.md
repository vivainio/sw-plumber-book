---
icon: lucide/hammer
---

# Make

Make is from 1976, predates every other tool in this book by decades, and is
still the thing most of them shell out to somewhere underneath — Autotools
generates makefiles, Linux kernel builds run on Make, and no small number of
"just run this script" project READMEs are secretly `make`-shaped. The model
it introduced — declare files, declare what they depend on, run a recipe
only when the dependency is newer than the target — is the ancestor of every
incremental build system that came after it, including ones that would never
admit the resemblance.

## The core model: targets, prerequisites, recipes

A makefile is a set of rules. Each rule has a target, a list of
prerequisites, and a recipe:

```makefile
target: prerequisite1 prerequisite2
	recipe line 1
	recipe line 2
```

Read literally: to build `target`, `prerequisite1` and `prerequisite2` must
exist and be up to date first; then, only if `target` doesn't exist or is
older than either prerequisite, run the recipe.

```makefile
app: main.o utils.o
	gcc -o app main.o utils.o

main.o: main.c utils.h
	gcc -c main.c

utils.o: utils.c utils.h
	gcc -c utils.c
```

Run `make app` and Make walks the dependency graph: `app` needs `main.o` and
`utils.o`, which need `main.c`/`utils.c` respectively. If none of the `.o`
files exist, both get compiled, then linked. Touch `utils.c` and run `make
app` again — only `utils.o` rebuilds, then the link step reruns, because
`app` is now older than the `utils.o` it depends on. Nothing recompiles
`main.o`, because `main.c` didn't change. This is the entire incremental
build story: **file modification timestamps compared along an explicit
dependency graph.**

That one sentence is worth sitting with, because it's also where Make's
sharp edges come from. Timestamps are a proxy for "did the meaningful
content change," and proxies leak: `touch main.c` with no content change
still triggers a rebuild; a build that writes an output file with a clock a
few seconds behind the source machine can make Make think a fresh build is
stale. Content-hash-based systems (Bazel, Nix) exist specifically to close
this gap — at the cost of needing to know inputs precisely enough to hash
them, which Make never required.

## Phony targets

Not every target is a file. `make clean`, `make test`, `make all` — none of
these produce a file literally named `clean`. If a file named `clean`
*did* exist in the directory, Make would see it, consider it "up to date"
(it has no prerequisites, so it's always up to date), and refuse to run the
recipe at all. `.PHONY` tells Make a target isn't a file, so it should
always run:

```makefile
.PHONY: clean test all

all: app

test: app
	./run_tests.sh

clean:
	rm -f *.o app
```

Nearly every real-world makefile has a `.PHONY` line near the top, and it's
there for exactly this reason — not style, a correctness fix for the
fact that Make's only notion of "already done" is a file on disk.

## Variables and pattern rules

Hand-writing a rule per source file doesn't scale. Pattern rules generalize
across files that share a suffix:

```makefile
CC = gcc
CFLAGS = -Wall -O2
OBJS = main.o utils.o

app: $(OBJS)
	$(CC) -o app $(OBJS)

%.o: %.c
	$(CC) $(CFLAGS) -c $< -o $@
```

`%.o: %.c` reads as "any `.o` file depends on the `.c` file with the same
stem." `$<` is the first prerequisite, `$@` is the target — Make's terse,
sed-like automatic variables. This is also usually the first place a
newcomer's eyes glaze over reading an unfamiliar makefile, and it's worth
learning deliberately rather than pattern-matching from examples, because
`$<`, `$@`, `$^`, and `$*` all mean different things and get shuffled
constantly in copy-pasted snippets.

## Recursive Make, and why it's controversial

Large C/C++ projects historically split into a makefile per directory, with
a top-level makefile that `cd`s into each and runs `make` again — "recursive
Make." Peter Miller's 1997 paper ["Recursive Make Considered
Harmful"](https://accu.org/journals/overload/14/71/miller_2004/) is the
canonical argument against it: each recursive invocation only sees its own
subtree, so Make's dependency graph is really N separate graphs stitched
together by directory traversal order, not one graph Make can reason about.
Two subdirectories that depend on each other in the wrong order will build
in the wrong order and Make will never notice, because from within either
invocation the other subtree doesn't exist. The fix (non-recursive Make,
`include`-ing every subdirectory's rules into one top-level makefile with
one real graph) is more correct and also, in most people's experience,
harder to read — which is exactly why recursive Make never went away
despite three decades of being told not to do it.

## Why this still matters if you never write a makefile

Every tool in the rest of this book reinvents some piece of what Make
already does: a graph of build steps, a way to skip steps whose inputs
haven't changed, and a way to force certain steps ("phony" ones) to always
run. Maven's fixed lifecycle, Gradle's task graph, and MSBuild's targets are
all answering the same underlying question Make asked first — what's the
minimum amount of work needed to bring this output up to date — with more
structure, more tooling, and (usually) content-aware invalidation instead of
timestamps. Reading a makefile closely once is the cheapest way to see that
underlying question clearly, before the next chapters bury it under XML and
DSLs.
