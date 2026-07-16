---
icon: lucide/home
---

# Software Plumbers

Every project runs on a layer nobody talks about at conferences: the
machinery that turns a source tree into a running artifact, and the
machinery that lets you see inside that artifact once it's running. Not
the framework, not the language, not the architecture — the plumbing.
Build tools like Make, Maven, Gradle, and npm. But also the debugger that
attaches to a live process, the profiler that samples its stack, the
linker that resolves its symbols, and the kernel interfaces underneath
all of it. Invisible when it works, and the only thing anyone thinks
about when it doesn't.

This book is about that layer, broadly. Not "getting started" tutorials —
every tool covered here already has those — but the parts that actually
cause outages and confusion: why a dependency resolves to a version
nobody asked for, how a debugger actually stops a running process, what a
profiler's sampling rate trades away, why a clean build behaves
differently from an incremental one, why a lockfile exists at all. Each
chapter picks one tool or mechanism and goes down to it, with real
examples instead of hand-waved diagrams.

It grows one topic at a time, not from a fixed curriculum. Topic requests
are welcome [in the issues](https://github.com/vivainio/sw-plumber-book/issues).

## Chapters

### Build systems

- [Make](make/index.md) — targets, prerequisites, and recipes; why `.PHONY`
  exists; and why every other tool on this list still shells out to it, or
  reinvents it, somewhere underneath.
- [Maven](maven/index.md) — the POM as a declarative build description, the
  fixed lifecycle, the reactor and multi-module builds, and why "plugin
  goals" are the only things that actually run.
- [Gradle](gradle/index.md) — tasks and the task graph, the Groovy/Kotlin
  DSL, incremental builds and the build cache, the daemon, and where it
  parts ways with Maven's fixed lifecycle.
- [Bazel](bazel/index.md) — hermetic builds as the explicit goal,
  actions instead of tasks, sandboxing that enforces declared
  dependencies instead of trusting them, and why the ceremony is the
  point.
- [MSBuild](msbuild/index.md) — the `.csproj` file as an MSBuild project,
  targets/tasks/items/properties, SDK-style projects, and why `dotnet build`
  is MSBuild wearing a friendlier CLI.
- [npm, pnpm, and Yarn](npm/index.md) — `package.json` scripts as the
  smallest possible build tool, `node_modules` resolution, lockfiles, and
  why pnpm and Yarn exist when npm already shipped first.
- [Rust and Cargo](rust-cargo/index.md) — one tool as build system, package
  manager, and task runner; requirements vs. the lockfile; profiles,
  fingerprinting, workspaces, and why feature flags are additive by design.
- [Python Packaging](python-packaging/index.md) — eggs versus wheels, why
  the wheel filename is a compatibility contract, manylinux and
  `auditwheel`, and building native extensions with setuptools and maturin.
- [Dependency Resolution & SemVer](dependency-resolution/index.md) — version
  ranges, the diamond dependency problem, and why semantic versioning is a
  promise between humans, not a guarantee a resolver can enforce.
- [CI/CD Pipelines](cicd-pipelines/index.md) — why GitHub Actions and
  Jenkins are orchestration, not build systems, and what breaks when a
  pipeline reimplements logic that belongs in the build file.

### Runtime & process internals

- [Debuggers](debuggers/index.md) — how `ptrace` and its OS-specific
  cousins let one process pause and inspect another, how a breakpoint is
  really a rewritten instruction, and why DWARF/PDB debug info is what
  turns a bare address back into `file.c:42`.
- [Containers](containers/index.md) — namespaces, cgroups, and union
  filesystems as the actual kernel primitives underneath "container,"
  why they're not lightweight VMs, and where Docker, containerd, and
  `runc` each fit in the stack.

More chapters are planned here — profilers, linkers/loaders, and the
other OS primitives underneath them. See the open issues for what's
queued.
