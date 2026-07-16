---
icon: lucide/workflow
---

# CI/CD Pipelines

A GitHub Actions workflow or a Jenkinsfile looks, superficially, like a
build tool — it has steps, it runs in order, some steps depend on others
finishing first. It isn't one. GitHub Actions and Jenkins are
**orchestration**: they decide *when* and *where* to run commands (which
event triggers it, which runner or agent executes it, what runs in parallel
across machines, what artifacts move between jobs) and they call out to the
actual build tool — Maven, Gradle, MSBuild, npm — to do the part that
turns source into an artifact. Confusing the two layers is the single most
common structural mistake in CI configuration, and it's worth stating the
boundary plainly before looking at either tool.

## The dependency graph is real — it's just made of jobs, not files

A GitHub Actions workflow's `needs` key is the same idea as Make's
prerequisites or Gradle's `dependsOn`, one level up — instead of "this file
depends on that file," it's "this job depends on that job's job finishing
first, potentially on an entirely different machine":

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: mvn package

  test:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: mvn test

  deploy:
    needs: [build, test]
    runs-on: ubuntu-latest
    steps:
      - run: ./deploy.sh
```

`test` and `deploy` waiting on `build` looks identical in shape to a
Makefile's target graph, but the mechanism is fundamentally different: each
job is a fresh, isolated environment (a new container or VM) with no shared
filesystem by default. `mvn package` in the `build` job produces a jar on
that runner's disk — and it's gone the moment the job ends, unless
explicitly carried forward.

## Artifacts and caches solve a problem Make never had

Because jobs run on genuinely separate machines, anything one job produces
that a later job needs has to move explicitly:

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - run: mvn package
      - uses: actions/upload-artifact@v4
        with:
          name: app-jar
          path: target/*.jar

  deploy:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - uses: actions/download-artifact@v4
        with:
          name: app-jar
      - run: ./deploy.sh app.jar
```

`upload-artifact`/`download-artifact` is the CI-level equivalent of a build
tool's own output directory — except it has to be spelled out explicitly,
because there's no shared disk connecting the two jobs the way there's a
shared `target/` directory between two phases in the same local Maven
invocation. This is also where a subtle failure mode lives: a workflow that
runs `mvn package` twice — once in `build`, redundantly again in `test`
because someone forgot the artifact already exists — isn't just wasting
time, it's a build tool re-resolving dependencies and recompiling on a
fresh runner with a cold cache, often taking as long as the original build
did.

Dependency caching (`actions/cache`, or Gradle/Maven's own remote cache
support) exists to blunt exactly this cost — restoring a previously saved
`~/.m2/repository` or `~/.gradle/caches` onto a fresh runner so a cold CI
machine doesn't re-download the entire dependency tree from the internet on
every single run. It's solving the same "avoid redoing work that's already
been done" problem Make and Gradle solve locally, just one layer up, across
runners that share nothing by default.

## The anti-pattern: build logic leaking into the pipeline

The failure mode worth naming explicitly: conditional logic, version
bumping, or packaging steps that belong in the `pom.xml`, `build.gradle`,
or `.csproj` instead get written directly into workflow YAML or a
Jenkinsfile's Groovy — because it's faster to add an `if` to a pipeline
step than to properly express it in the build tool's own model. This works
right up until someone tries to run the build locally to reproduce a CI
failure, and discovers the actual packaging step only exists inside CI,
never runs on a laptop, and is untestable outside of pushing a commit and
waiting for a runner. A build that only works inside its pipeline isn't
reproducible — the entire point of a pipeline is to *automate* running the
same build a developer can run locally, not to *be* the build.

```yaml
# Anti-pattern: version logic that only exists in CI
- run: |
    if [ "$GITHUB_REF" == "refs/heads/main" ]; then
      mvn versions:set -DnewVersion=1.0.${{ github.run_number }}
    fi
    mvn package
```

The fix isn't "don't do versioning in CI" — it's keeping the *decision*
(what triggered this, which branch, which run) in the pipeline, while the
*mechanism* (how a version number actually gets applied to the POM) stays
in a Maven plugin invocation that a developer could run with the same
inputs locally and get the identical result.

## Matrix builds: the orchestration layer's version of a task graph fan-out

```yaml
jobs:
  test:
    strategy:
      matrix:
        os: [ubuntu-latest, windows-latest, macos-latest]
        java: [17, 21]
    runs-on: ${{ matrix.os }}
    steps:
      - uses: actions/setup-java@v4
        with:
          java-version: ${{ matrix.java }}
      - run: mvn test
```

This expands into six independent jobs (3 OSes × 2 Java versions), each
running the identical `mvn test` invocation with a different environment —
a fan-out no build tool covered in this book handles on its own, because it
isn't a build concern, it's a "run this build N times under different
conditions and report on all of them" concern. That's the clearest
one-sentence description of what CI orchestration actually adds on top of
a build tool: not a better build, but running the same build repeatedly,
in parallel, across environments a single machine couldn't represent at
once, and aggregating the results.

## Jenkins vs. GitHub Actions: same job, different distribution model

Jenkins predates GitHub Actions by over a decade and runs on
self-hosted agents you provision and maintain; a Jenkinsfile is Groovy,
giving it the same real-language flexibility (and the same
readability cost) that Gradle's DSL has over Maven's XML. GitHub Actions
trades that self-hosting requirement for GitHub-managed runners and a
large marketplace of reusable `uses:` steps, at the cost of being
inherently tied to GitHub as the source host. Neither difference changes
the underlying model this chapter is about — both are orchestrating calls
out to Maven, Gradle, MSBuild, or npm to do the actual building, and both
have the identical failure mode available if build logic gets written into
the orchestration layer instead of the build tool it's supposed to be
invoking.
