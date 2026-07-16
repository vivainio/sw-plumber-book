---
icon: lucide/link-2
---

# Dependency Resolution & SemVer

Every tool in this book — Maven, Gradle, MSBuild's NuGet integration, npm —
solves a version of the same problem: given a project that declares
dependencies, and dependencies of dependencies going arbitrarily deep, pick
one consistent set of exact versions to actually use. The algorithms differ
in detail, but the underlying tension is identical everywhere, and it's
worth seeing as one problem rather than four unrelated ecosystem quirks.

## Version ranges describe intent, not a decision

`^1.2.0` (npm), `[1.2.0,2.0.0)` (Maven), or a NuGet floating version all say
the same thing: "any version compatible with 1.2.0, per whatever
compatibility means in this ecosystem's convention" — not "install exactly
1.2.0." This is deliberate: a project depending on an exact pinned version
of everything, transitively, would make it nearly impossible for two
libraries to ever share a dependency, because any difference in their
pinned sub-dependencies would be an unresolvable conflict rather than a
range the resolver can satisfy from either side. Ranges exist specifically
to give the resolver room to reconcile what different parts of the tree are
asking for.

## The diamond dependency problem

Say `App` depends on `LibraryA` and `LibraryB`. `LibraryA` needs
`Utils >= 2.0`, `LibraryB` needs `Utils >= 1.5, < 2.0`. Drawn out, the
dependency graph looks like a diamond — `App` at the top, `LibraryA` and
`LibraryB` in the middle, both pointing at `Utils` underneath — and there's
no single version of `Utils` that satisfies both constraints. This isn't a
bug in any particular resolver; it's a structural consequence of letting
independent library authors each pick their own version constraints without
coordinating with each other. Every ecosystem's resolver has to pick a
strategy for this exact situation, and the strategy is usually where
ecosystems diverge most visibly:

- **Nearest wins** (Maven): the dependency closer to your own POM in the
  tree is chosen, regardless of whether it's the newer or older version.
  Deterministic, but "nearest" is a proxy for "correct" the same way
  timestamps are a proxy for "changed" in Make — usually right, sometimes
  not, and not obviously so from reading the POM.
- **Newest wins** (Gradle default, npm's flattening in practice): the
  highest version anywhere in the tree is selected. Usually the more
  compatible choice in practice (semver assumes newer minor/patch versions
  are backward compatible), but "usually" is doing real work in that
  sentence.
- **Multiple versions coexist** (npm/Node, because `node_modules` nesting
  allows it; Java class loaders under OSGi, for the same reason): the
  conflict is avoided entirely by letting both versions exist
  simultaneously, isolated from each other. This works when the runtime
  supports loading two versions of the same logical module without them
  colliding — something the JVM's default classpath model, notably, does
  *not* support, which is exactly why Java tooling had to invent
  nearest/newest-wins strategies that Node never needed for its own
  dependency tree.

None of these strategies produce a version that's provably correct for
both `LibraryA` and `LibraryB` — they produce *a* version, deterministically,
and it's on the build to actually test that the choice works. This is the
gap between "the build resolved successfully" and "the resolved versions
are actually compatible at runtime" — the first is a graph algorithm
guarantee, the second is not, and confusing them is how a
`NoSuchMethodError` or `ClassNotFoundException` shows up at runtime despite
a completely clean build.

## Semantic versioning is a promise, not an enforced contract

SemVer (`MAJOR.MINOR.PATCH`) says: increment `PATCH` for backward-compatible
bug fixes, `MINOR` for backward-compatible new features, `MAJOR` for
breaking changes. Every resolver's "compatible range" logic (`^1.2.0` means
"anything before `2.0.0`") is built entirely on trusting that publishers
follow this rule correctly. Nothing mechanically enforces it — no registry
runs your test suite against the previous major version before allowing a
patch release, and there's no way for a resolver to detect "this patch
release actually changed public behavior" without literally running the
consuming code against it. A library author fixing what they consider a bug
but changing observable behavior in the process, shipped as a patch bump,
is a semver violation that no tooling in the chain will catch before it
reaches a consumer's build — the entire system runs on publisher discipline,
not verification.

## Lockfiles: pinning the resolution, not changing the ranges

`package-lock.json`, `Gemfile.lock`, `poetry.lock`, and (less
universally, since Maven doesn't ship one natively — the closest
equivalent is a `dependencyManagement`-pinned BOM or the Maven Enforcer
plugin's dependency convergence rule) similar mechanisms all solve the same
problem: `package.json` describes acceptable ranges, but a build needs an
*exact*, reproducible set of versions, or "works on my machine" becomes
"works on my machine, this week." A lockfile is the output of running the
resolver once and freezing the answer — the ranges in `package.json` are
still what's *allowed*, the lockfile is what's *actually installed*, until
someone deliberately regenerates it. Ecosystems that skip this step
entirely (raw `pip install -r requirements.txt` with unpinned versions
being the classic example) reintroduce the diamond dependency problem
freshly on every single install, on every machine, indefinitely.

## Where this actually breaks builds in practice

- **A transitive dependency's patch release breaks something**, despite
  following semver correctly on paper — the "bug fix" changed behavior
  something downstream was silently relying on. This is not a tooling
  failure; it's the gap between "backward compatible" as a promise and as
  a verified property.
- **Two ecosystems' resolvers disagree about the same logical conflict** in
  a polyglot repo — a Gradle build and an npm build in the same monorepo
  each resolving a shared transitive dependency (say, a protobuf runtime
  pulled in on both the JVM and Node side) to different versions, with no
  single tool positioned to notice the mismatch because neither resolver
  is aware the other ecosystem exists.
- **A resolver silently picks a version nobody explicitly asked for**, and
  the first sign is a runtime error, not a build failure — which is exactly
  why `mvn dependency:tree`, `gradle dependencies`, and `npm ls` all exist:
  the declared dependencies in a manifest are an input to resolution, not a
  description of the result, and the only way to see the result is to ask
  the resolver directly.
