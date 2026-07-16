---
icon: lucide/git-branch
---

# Gradle

Gradle showed up in 2007 as the answer to a specific complaint about Maven:
the fixed lifecycle is great until you need to do something it didn't
anticipate, and then you're bolting plugin executions onto phases that were
never designed for your use case. Gradle's pitch was to keep Maven's
dependency management and convention-based defaults, but replace the fixed
lifecycle with a real, general-purpose build graph — arbitrary tasks, with
arbitrary dependencies between them, defined in a real programming language
instead of XML. It's the closest thing in this book to Make's model, wearing
JVM ecosystem clothing.

## Tasks are the unit of everything

Where Maven has phases-with-goals-bound-to-them, Gradle just has tasks —
and a task can depend on any other task, forming a graph you can actually
see:

```kotlin
tasks.register("hello") {
    doLast {
        println("Hello from Gradle")
    }
}

tasks.register("goodbye") {
    dependsOn("hello")
    doLast {
        println("Goodbye")
    }
}
```

`gradle goodbye` runs `hello` first, because `dependsOn` says so — not
because of a fixed ordering built into Gradle itself. `gradle tasks` lists
every task available in a project; `gradle :app:dependencies` and the
`--dry-run` flag (which prints the execution plan without running anything)
are the two commands worth reaching for before assuming what a build will
do. The Java/Kotlin/Android plugins layer familiar-looking tasks on top of
this — `compileJava`, `test`, `build`, `assemble` — but underneath, they're
just tasks with `dependsOn` edges like any other, which means you can insert
your own task into that graph the same way the plugin authors did, not by
fighting a fixed lifecycle.

## Groovy and Kotlin DSL: it's a real language, which cuts both ways

A `build.gradle.kts` file is Kotlin. This is Gradle's biggest structural
departure from Maven's POM: because it's a real language, you can write
actual logic — loops, conditionals, functions — directly in the build file:

```kotlin
val supportedVersions = listOf(11, 17, 21)

tasks.register("printVersions") {
    doLast {
        supportedVersions.forEach { println("Supports Java $it") }
    }
}
```

That expressiveness is exactly what people wanted after fighting Maven's
declarative-only POM — and exactly what makes a large, poorly maintained
Gradle build genuinely hard to read cold, because "what does this build
actually do" now requires evaluating arbitrary code rather than reading a
static description. The Kotlin DSL over the older Groovy DSL trades a bit of
that flexibility back for IDE autocomplete and compile-time checking of the
build script itself, which is why most new Gradle projects default to it
now.

## Incremental builds: inputs and outputs, not timestamps

This is where Gradle actually improves on Make's model rather than just
restating it. A Make target is stale if it's older than its prerequisite —
a timestamp comparison. A Gradle task declares its inputs and outputs
explicitly, and Gradle hashes them:

```kotlin
tasks.register("generateReport") {
    inputs.file("data.csv")
    outputs.file("report.txt")
    doLast {
        file("report.txt").writeText(file("data.csv").readText().uppercase())
    }
}
```

Run this task twice with `data.csv` unchanged and Gradle skips it entirely
— `UP-TO-DATE` — because the hash of the declared inputs matches the hash
from last time, regardless of what the file's modification timestamp says.
Touch the file without changing its content (the exact case that fools
Make) and Gradle correctly does nothing, because it's comparing content
hashes, not mtimes. This is also why an undeclared input is a real bug
class specific to Gradle: if a task reads an environment variable or a file
it never declared via `inputs`, Gradle has no way to know that value
changed, and will happily report `UP-TO-DATE` on a build that's actually
stale.

## The build cache goes further: it's not even about *this* machine

Incremental builds skip work within one build directory. The **build
cache** (opt-in, `org.gradle.caching=true`) goes further: task outputs are
keyed by a hash of their inputs and stored in a cache that can be shared
across machines. If a teammate — or a CI runner — already built a task with
the identical input hash, the output is fetched from cache instead of
recomputed, even though this exact machine has never run that task before.
This only works because of the same discipline incremental builds require:
a task's declared inputs have to be the *complete* set of things that affect
its output, or two different results end up sharing a cache key and one
machine gets someone else's stale output.

## The daemon: paying JVM startup cost once, not every build

The Gradle daemon is a long-lived background JVM process that keeps class
metadata and a warmed-up JIT between builds, instead of every `gradle`
invocation cold-starting a fresh JVM. This is the single biggest reason
Gradle's second build of a session feels dramatically faster than its
first — it's not incrementality, it's avoiding JVM startup and
classloading twice. It's also the source of a specific, recurring debugging
dead end: a build behaving differently than expected after changing a
plugin version or an environment variable, where the fix is `gradle
--stop` to kill the daemon and force a clean process, because the running
daemon cached something (a plugin class, a stale environment snapshot) from
before the change.

## Gradle vs. Maven: same dependency resolution, different everything else

Under the hood, Gradle and Maven resolve dependencies from the same
repositories using compatible logic — GAV coordinates, transitive
resolution, version conflict handling (Gradle defaults to newest-wins,
Maven to nearest-wins, which is worth knowing if a multi-module migration
between them ever changes a resolved version silently). The real difference
is the build-definition model: Maven's fixed lifecycle trades flexibility
for every project looking the same shape; Gradle's task graph trades that
uniformity for the ability to model builds Maven's lifecycle was never
designed to express — Android's variant builds (debug/release ×
flavor combinations, each needing its own task subgraph) are the case study
that made this tradeoff worth it for an entire ecosystem, which is why
Android builds are Gradle-only in practice, not a Maven plugin.
