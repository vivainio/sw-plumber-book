---
icon: lucide/home
---

# Software Plumbers

Every project has a layer nobody talks about at conferences: the thing that
turns a source tree into a running artifact. Not the framework, not the
language, not the architecture — the build. Make, Maven, Gradle, MSBuild,
npm scripts, and the CI pipeline that stitches them together. It's the
plumbing: invisible when it works, and the only thing anyone thinks about
when it doesn't.

This book is about that layer. Not "getting started" tutorials — every tool
covered here already has those — but the parts that actually cause outages
and confusion: why a dependency resolves to a version nobody asked for, why
a clean build behaves differently from an incremental one, why the same
`pom.xml` behaves differently on two machines, why a lockfile exists at
all. Each chapter picks one tool and goes down to the mechanism, with real
project files instead of hand-waved diagrams.

It grows one tool at a time, not from a fixed curriculum. Topic requests
are welcome [in the issues](https://github.com/vivainio/sw-plumber-book/issues).

## Chapters

- [Make](make/index.md) — targets, prerequisites, and recipes; why `.PHONY`
  exists; and why every other tool on this list still shells out to it, or
  reinvents it, somewhere underneath.
- [Maven](maven/index.md) — the POM as a declarative build description, the
  fixed lifecycle, the reactor and multi-module builds, and why "plugin
  goals" are the only things that actually run.
- [Gradle](gradle/index.md) — tasks and the task graph, the Groovy/Kotlin
  DSL, incremental builds and the build cache, the daemon, and where it
  parts ways with Maven's fixed lifecycle.
- [MSBuild](msbuild/index.md) — the `.csproj` file as an MSBuild project,
  targets/tasks/items/properties, SDK-style projects, and why `dotnet build`
  is MSBuild wearing a friendlier CLI.
- [npm, pnpm, and Yarn](npm/index.md) — `package.json` scripts as the
  smallest possible build tool, `node_modules` resolution, lockfiles, and
  why pnpm and Yarn exist when npm already shipped first.
- [Dependency Resolution & SemVer](dependency-resolution/index.md) — version
  ranges, the diamond dependency problem, and why semantic versioning is a
  promise between humans, not a guarantee a resolver can enforce.
- [CI/CD Pipelines](cicd-pipelines/index.md) — why GitHub Actions and
  Jenkins are orchestration, not build systems, and what breaks when a
  pipeline reimplements logic that belongs in the build file.
