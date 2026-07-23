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

In a real project, use the checked-in wrapper rather than whichever Gradle
happens to be installed on the machine:

```console
$ ./gradlew goodbye --dry-run
:hello SKIPPED
:goodbye SKIPPED

$ ./gradlew goodbye

> Task :hello
Hello from Gradle

> Task :goodbye
Goodbye
```

`SKIPPED` here does not mean the task would be omitted from a real build.
`--dry-run` is showing the selected tasks in execution order while
deliberately skipping their actions. The leading colon is the task's
fully-qualified path: in a multi-project build, `:app:test` and
`:library:test` are different tasks even though both have the short name
`test`.

## Three phases: the source of many Gradle surprises

A Gradle invocation has three conceptually separate phases:

1. **Initialization** discovers the build and its included projects, starting
   with `settings.gradle.kts`.
2. **Configuration** evaluates build scripts and constructs/configures the task
   graph.
3. **Execution** runs the actions of the selected tasks.

Code in a task's configuration block is not the task's work. It runs while
Gradle is configuring the build. The work belongs in `doFirst`, `doLast`, or
an action on a custom task type:

```kotlin
println("configuring the build")

tasks.register("demonstrate") {
    println("configuring :demonstrate")
    doLast {
        println("executing :demonstrate")
    }
}
```

```console
$ ./gradlew demonstrate
configuring the build
configuring :demonstrate

> Task :demonstrate
executing :demonstrate
```

This distinction explains the classic "why did that code run when I asked
for a different task?" bug: a file read, network request, or `println` placed
directly in configuration code can happen before Gradle has executed any
task. `tasks.register` supports lazy task creation, but its configuration
still needs to remain configuration rather than hidden work.

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

The status words in normal output tell you *why* work did or did not happen:

```console
$ ./gradlew generateReport
> Task :generateReport

$ ./gradlew generateReport
> Task :generateReport UP-TO-DATE

$ ./gradlew generateReport --rerun-tasks
> Task :generateReport
```

Other common statuses are `NO-SOURCE` (the task had no source files),
`SKIPPED` (for example, an `onlyIf` predicate rejected it), and `FROM-CACHE`
(Gradle restored the outputs rather than executing the task). If a task is
unexpectedly up to date, `--info` is the first useful level of extra output.
If it reruns unexpectedly, look for an input value or input file that Gradle
says changed; volatile generated files, absolute paths, and environment
values accidentally modeled as inputs are frequent causes.

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
classloading twice. `./gradlew --status` lists compatible daemons and
`./gradlew --stop` stops them. Restarting the daemon is a useful diagnostic
when investigating process-level state, file locks, or memory pressure, but
it should not be the routine fix for an incorrectly modeled build: if a task
uses a value that affects its output, that value belongs in the task's
declared inputs.

## Dependencies live in configurations

The word "dependency" is overloaded in Gradle. `dependsOn` creates an edge
between *tasks*. Library dependencies such as `org.slf4j:slf4j-api` live in
named **configurations**—sets of dependencies with a purpose such as
compilation or runtime:

```kotlin
dependencies {
    implementation("org.slf4j:slf4j-api:2.0.17")
    testImplementation("org.junit.jupiter:junit-jupiter:5.13.4")
}
```

The Java plugin supplies names such as `implementation`,
`testImplementation`, `compileClasspath`, and `runtimeClasspath`.
`implementation` is where you declare a direct dependency;
`compileClasspath` is a resolvable view Gradle derives from declarations.
Keeping those roles separate matters in custom build logic: a bucket people
declare into is not necessarily a configuration you should resolve.

When the selected version is surprising, inspect the resolved graph rather
than reading the build file harder:

```console
$ ./gradlew dependencies --configuration runtimeClasspath

$ ./gradlew dependencyInsight \
    --dependency slf4j-api \
    --configuration runtimeClasspath
org.slf4j:slf4j-api:2.0.17
   Selection reasons:
      - By conflict resolution: between versions 2.0.17 and 1.7.36
```

`dependencies` answers "what is in this configuration?" while
`dependencyInsight` answers "why did this particular component and version
win?" That second question is usually the useful one in a conflict.

## Multi-project builds: paths make the graph unambiguous

`settings.gradle.kts` defines which projects participate:

```kotlin
rootProject.name = "shop"
include("app", "pricing")
```

One project can depend on another's produced library:

```kotlin
// app/build.gradle.kts
dependencies {
    implementation(project(":pricing"))
}
```

This is not the same as `tasks.named("compileJava") {
dependsOn(":pricing:compileJava") }`. A project dependency carries the
artifact and usage information Gradle needs and lets the Java plugins infer
the appropriate task edges. A hand-written task dependency merely orders
two tasks; it does not put `pricing` on `app`'s classpath.

## A compact debugging sequence

When a Gradle build is opaque, narrow the question in this order:

```console
./gradlew projects
./gradlew tasks
./gradlew :app:build --dry-run
./gradlew :app:build --info
./gradlew dependencyInsight --dependency NAME --configuration CONFIGURATION
```

The first two establish the build's vocabulary, the dry run reveals task
selection and ordering, `--info` explains incremental decisions, and
`dependencyInsight` handles the separate problem of library resolution.
`./gradlew properties` and `./gradlew :app:properties` are useful when the
mystery is a project property or plugin-provided convention instead.

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
