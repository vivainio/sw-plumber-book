---
icon: lucide/feather
---

# Maven

Maven's pitch, in 2004, was radical compared to what came before it (mostly
Ant, which was Make with XML syntax and no built-in opinions): stop writing
build logic, and instead *declare* what your project is — its coordinates,
its dependencies, its packaging type — and let a fixed lifecycle figure out
what to run. Convention over configuration, applied to builds. Two decades
later that fixed lifecycle is simultaneously Maven's best feature and the
reason every non-trivial Maven build ends up full of `<plugin>` blocks
fighting the thing that was supposed to remove them.

## The POM is a description, not a script

A `pom.xml` doesn't say "compile, then test, then package." It says what
the project *is*:

```xml
<project>
  <modelVersion>4.0.0</modelVersion>
  <groupId>com.example</groupId>
  <artifactId>my-app</artifactId>
  <version>1.0.0</version>
  <packaging>jar</packaging>

  <dependencies>
    <dependency>
      <groupId>org.junit.jupiter</groupId>
      <artifactId>junit-jupiter</artifactId>
      <version>5.10.0</version>
      <scope>test</scope>
    </dependency>
  </dependencies>
</project>
```

`groupId:artifactId:version` (a "GAV" coordinate) is how Maven names every
artifact, everywhere — your own project, and everything it depends on. This
one is enough to build a working JAR: `mvn package` compiles `src/main/java`,
runs anything in `src/test/java`, and produces
`target/my-app-1.0.0.jar` — no build script written, because `packaging =
jar` already told Maven which lifecycle bindings apply.

## The lifecycle is fixed; plugins fill in the goals

Maven has exactly three built-in lifecycles (`default`, `clean`, `site`),
and `default` is the one that matters day to day. It's a fixed, ordered list
of phases:

```
validate → compile → test → package → verify → install → deploy
```

Running `mvn install` doesn't just run `install` — it runs every phase up
to and including it, in order. This is the part that trips people coming
from Make or Gradle: **you cannot add a new phase.** What you *can* do is
bind a plugin goal to an existing phase. `mvn package` produces a jar
because the `jar` packaging type binds `maven-jar-plugin:jar` to the
`package` phase, by convention, with zero configuration. Want to run a
custom script before tests run? It doesn't get its own phase — it gets
bound to `test-compile` or `process-test-classes`, whichever existing phase
is the closest fit:

```xml
<build>
  <plugins>
    <plugin>
      <groupId>org.codehaus.mojo</groupId>
      <artifactId>exec-maven-plugin</artifactId>
      <executions>
        <execution>
          <phase>generate-sources</phase>
          <goals><goal>exec</goal></goals>
        </execution>
      </executions>
    </plugin>
  </plugins>
</build>
```

This is the tradeoff in one example: every Maven build looks the same
shape from the outside (`compile`, `test`, `package` mean the same thing in
every Maven project ever written), but customizing anything beyond what a
plugin already anticipated means wedging your logic into someone else's
phase, not writing your own step.

## Dependency scopes control the classpath, not just "is it included"

A `<scope>` is not metadata — it changes which classpath a dependency lands
on and whether it's transitive:

| Scope | Compile classpath | Test classpath | Runtime | Transitive to consumers |
|---|---|---|---|---|
| `compile` (default) | yes | yes | yes | yes |
| `provided` | yes | yes | no | no |
| `runtime` | no | yes | yes | yes |
| `test` | no | yes | no | no |

`provided` is the one people reach for without fully internalizing what it
means: it says "this is on the classpath at compile time, but something
*else* (a servlet container, a Spark cluster, an `-agentlib` jar) provides
it at runtime, so don't bundle it." Get this wrong on a shaded/fat jar and
you either bloat the artifact with something the runtime already supplies,
or ship a jar that throws `NoClassDefFoundError` in production because a
`test`-scoped dependency silently didn't make it into the runtime
classpath.

## Transitive dependencies and "nearest wins"

If `my-app` depends on `library-a`, and `library-a` depends on
`commons-lang3:3.9`, Maven pulls `commons-lang3` in transitively without
being asked. This is convenient until two libraries disagree: if `my-app`
also depends on `library-b`, which needs `commons-lang3:3.12`, Maven has to
pick one version for the whole build — Java doesn't support two versions of
the same class on one classpath. Maven's rule is **nearest wins**: whichever
dependency is closer in the tree to your own POM (fewest hops) is used, and
ties go to declaration order in the POM. This is deterministic, but not
obviously so from reading the POM alone — `mvn dependency:tree` is the
command that actually answers "which version am I getting and why," and
reaching for it should be reflexive the moment two libraries need
overlapping dependencies.

```
mvn dependency:tree -Dverbose
```

The `-Dverbose` flag is what surfaces the versions that *lost* the
resolution and why, not just the winner — without it the tree only shows
what got resolved, which is the wrong direction to debug from when a
runtime `NoSuchMethodError` says some class changed shape between versions.

## The reactor: multi-module builds

A parent POM with `<modules>` turns a directory of related projects into one
build:

```xml
<project>
  <groupId>com.example</groupId>
  <artifactId>my-app-parent</artifactId>
  <packaging>pom</packaging>
  <modules>
    <module>core</module>
    <module>api</module>
    <module>web</module>
  </modules>
</project>
```

Running `mvn install` from the parent invokes the **reactor**, which reads
every module's POM, builds a dependency graph between the modules themselves
(if `web` depends on `api`, `api` must build first), and topologically sorts
the build order — this is Make's dependency-graph idea again, just applied
at the level of whole modules instead of individual files. `mvn install
-pl web -am` builds only `web` and everything it transitively depends on
(`-am` = "also make") — the reactor equivalent of Make only rebuilding the
stale part of the graph, useful once a multi-module build gets big enough
that a full `mvn install` takes minutes instead of seconds.

## Where people actually get burned

- **`mvn clean install` as a reflex.** The fixed lifecycle makes Maven look
  deterministic, but plugins can and do leave stale state (annotation
  processor output, generated sources) that a partial rebuild doesn't
  regenerate correctly. `clean` deletes `target/` first — it's the
  timestamp-cache-busting equivalent of `touch`-ing every file in a Make
  build, and it becomes habitual precisely because Maven's incrementality is
  weaker than Make's or Gradle's.
- **The `<dependencyManagement>` vs `<dependencies>` split.** A parent POM's
  `<dependencyManagement>` block declares *versions*, not *dependencies* —
  it doesn't pull anything in by itself. A child module still needs its own
  `<dependency>` entry (without a version) to actually get the artifact.
  Forgetting this is the single most common "why isn't this on my
  classpath" question in multi-module Maven projects.
- **Snapshot versions resolving differently across machines.** A `-SNAPSHOT`
  dependency is mutable — Maven re-resolves it from the repository on a
  schedule rather than treating it as immutable once downloaded, which is
  exactly the property a reproducible build needs *not* to have. It's fine
  for local development against a library you're actively changing, and a
  liability the moment it survives into a release build.
