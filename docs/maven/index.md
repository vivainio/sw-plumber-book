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
source of its characteristic complexity: when a build doesn't fit the
conventions, customization arrives as `<plugin>` executions bound to the
nearest available phase.

## The POM is a description, not a script

A `pom.xml` doesn't say "compile, then test, then package." It says what
the project *is*:

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0">
  <modelVersion>4.0.0</modelVersion>
  <groupId>com.example</groupId>
  <artifactId>my-app</artifactId>
  <version>1.0.0</version>
  <properties>
    <maven.compiler.release>21</maven.compiler.release>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
  </properties>

  <dependencies>
    <dependency>
      <groupId>org.junit.jupiter</groupId>
      <artifactId>junit-jupiter</artifactId>
      <version>5.11.4</version>
      <scope>test</scope>
    </dependency>
  </dependencies>
</project>
```

`groupId:artifactId:version` (a "GAV" coordinate) is how Maven names every
artifact, everywhere — your own project, and everything it depends on. This
one is enough to build a working Java 21 JAR: `mvn package` compiles `src/main/java`,
runs anything in `src/test/java`, and produces
`target/my-app-1.0.0.jar` — no build script written, because `packaging =
jar` is the default and already tells Maven which lifecycle bindings apply.
The two properties are not required by the POM model, but leaving the Java
release and source encoding to whatever defaults happen to come with a
particular Maven installation is a poor basis for a reproducible build.

## Start with the Wrapper, not a machine-wide Maven

Most maintained projects check in the **Maven Wrapper**: `mvnw`,
`mvnw.cmd`, and a small `.mvn/wrapper/` directory. Run it as:

```sh
./mvnw verify          # Unix and macOS
mvnw.cmd verify        # Windows
```

The wrapper downloads and runs the Maven version chosen by the project.
That makes the Maven distribution part of the build definition instead of
an undocumented property of each developer laptop or CI image. A bare
`mvn verify` uses whichever Maven appears first on the current machine's
`PATH`; it is useful when exploring, but a repository that supplies
`./mvnw` is telling you which entry point is reproducible.

If a project does not have a wrapper yet, a current Maven installation can
generate one:

```sh
mvn wrapper:wrapper
```

The wrapper pins Maven, not Java. The JDK is a separate input, addressed
later with compiler settings and toolchains.

## The standard directory layout is part of the convention

The POM above contains no source paths because Maven assigns meanings to
well-known directories:

| Path | Meaning |
|---|---|
| `src/main/java` | production Java sources |
| `src/main/resources` | files copied onto the runtime classpath |
| `src/test/java` | test Java sources |
| `src/test/resources` | files copied onto the test classpath |
| `target/classes` | compiled production classes and resources |
| `target/test-classes` | compiled tests and test resources |
| `target/` | all disposable build output |

Putting a configuration file in `src/main/resources/com/example/app.properties`
therefore makes it available as
`com/example/app.properties` from the application classloader. Maven isn't
discovering its type or inspecting its contents; the resources plugin copies
the directory tree during `process-resources`.

You can override every path, but doing so spends one of Maven's main
advantages: a developer can open an unfamiliar repository and know where
the code, tests, resources, and outputs are before reading its POM.

## The Super POM and the effective POM

A minimal POM containing only the model version and GAV can still build,
test, and package a conventional project, which looks like magic until you
know that every POM implicitly extends a built-in **Super POM**, shipped
inside the Maven distribution itself (`org/apache/maven/model/pom-4.0.0.xml`
in `maven-model-builder.jar`). It's where `src/main/java`,
`src/test/java`, `target/` as the build directory, and Maven Central as
the default remote repository are declared.

Your POM, any `<parent>` chain above it, and the Super POM at the root of
that chain are merged — parent values filled in where the child didn't
override them, `${property}` references interpolated — into what Maven
actually builds from: the **effective POM**.

```sh
mvn help:effective-pom
```

```xml
<project>
  <modelVersion>4.0.0</modelVersion>
  <groupId>com.example</groupId>
  <artifactId>my-app</artifactId>
  <version>1.0.0</version>
  <packaging>jar</packaging>

  <build>
    <sourceDirectory>/home/user/my-app/src/main/java</sourceDirectory>
    <testSourceDirectory>/home/user/my-app/src/test/java</testSourceDirectory>
    <directory>/home/user/my-app/target</directory>
    <finalName>my-app-1.0.0</finalName>
    <plugins>
      <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-compiler-plugin</artifactId>
        <version>3.13.0</version>
      </plugin>
      <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-jar-plugin</artifactId>
        <version>3.4.1</version>
        <executions>
          <execution>
            <id>default-jar</id>
            <phase>package</phase>
            <goals><goal>jar</goal></goals>
          </execution>
        </executions>
      </plugin>
      <!-- ...surefire, install, deploy, resources, and clean, each with
           a pinned version and a default-* execution bound to its phase -->
    </plugins>
  </build>

  <repositories>
    <repository>
      <id>central</id>
      <url>https://repo.maven.apache.org/maven2</url>
    </repository>
  </repositories>

  <dependencies>
    <dependency>
      <groupId>org.junit.jupiter</groupId>
      <artifactId>junit-jupiter</artifactId>
      <version>5.11.4</version>
      <scope>test</scope>
    </dependency>
  </dependencies>
</project>
```

None of `sourceDirectory`, `finalName`, or the `central` repository appears
in the small POM above: those values came from the Super POM (or may be
overridden by a project parent). The `default-jar` execution has a different
origin: Maven's `jar` **packaging lifecycle mapping** binds `jar:jar` to
`package`. Lifecycle mappings are built into Maven separately from POM
inheritance, even though `help:effective-pom` presents the combined result.

That distinction gives "settings from nowhere" three common sources:

1. the Super POM and a project's `<parent>` chain;
2. active profiles merged into that model;
3. lifecycle bindings selected by `<packaging>`.

`help:effective-pom` is still the best first view because the explicit file
on disk is deliberately the smallest representation, not the complete one.
Add `-Dverbose` when you want comments showing where effective elements
were declared:

```sh
./mvnw help:effective-pom -Dverbose
```

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
conventional Maven projects), but customizing anything beyond what a plugin
already anticipated means binding your logic to someone else's phase, not
writing your own step.

## Which lifecycle command should you actually run?

The phase names form a ladder, so choose the highest rung containing the
checks you need:

| Command | What it is normally for |
|---|---|
| `./mvnw test` | compile and run unit tests |
| `./mvnw package` | also produce the JAR/WAR |
| `./mvnw verify` | also run integration checks and quality gates |
| `./mvnw install` | also copy project artifacts into local `~/.m2` |
| `./mvnw deploy` | also publish to a configured remote repository |

`verify` is the useful default for CI and for checking a change locally.
`install` is not a stronger verification phase; its distinctive effect is
mutating the local repository so another, separately invoked build can
resolve this project's artifact. Inside one multi-module reactor Maven can
pass artifacts between modules without first installing them, so reflexive
`clean install` often does more work and leaves more state than needed.

`clean` belongs to a separate lifecycle. `./mvnw clean verify` first deletes
`target/`, then runs the default lifecycle through `verify`. It is valuable
when diagnosing stale generated output, but using it on every edit discards
all incremental work and can hide plugins that fail to describe or update
their outputs correctly.

Two similar-looking test skip switches have deliberately different effects:

```sh
./mvnw package -DskipTests          # compile tests, do not run them
./mvnw package -Dmaven.test.skip    # do not compile or run them
```

The first is usually less surprising because test sources still have to
compile.

## Unit tests and integration tests occupy different phases

The default `test` phase invokes **Surefire**, conventionally discovering
names such as `*Test`. Integration tests usually use **Failsafe**, whose
goals deliberately straddle several phases:

```xml
<plugin>
  <groupId>org.apache.maven.plugins</groupId>
  <artifactId>maven-failsafe-plugin</artifactId>
  <version>3.5.2</version>
  <executions>
    <execution>
      <goals>
        <goal>integration-test</goal>
        <goal>verify</goal>
      </goals>
    </execution>
  </executions>
</plugin>
```

Failsafe conventionally discovers `*IT`. Its `integration-test` goal runs
the tests, while `verify` checks their results after any
`post-integration-test` cleanup has had a chance to run. This is why invoking
`./mvnw integration-test` directly is usually wrong: a server or container
started in `pre-integration-test` can be left running, and the build may not
reach the phase that turns test failures into a failed build. Invoke
`./mvnw verify`.

## Plugins are build dependencies, so pin their versions

Project dependencies become part of a project classpath. Build plugins are
separate executable dependencies loaded into Maven itself. Both can change
the result, so both need versions if a build is to remain stable across
Maven releases and machines:

```xml
<build>
  <plugins>
    <plugin>
      <groupId>org.apache.maven.plugins</groupId>
      <artifactId>maven-compiler-plugin</artifactId>
      <version>3.13.0</version>
      <configuration>
        <release>21</release>
      </configuration>
    </plugin>
    <plugin>
      <groupId>org.apache.maven.plugins</groupId>
      <artifactId>maven-surefire-plugin</artifactId>
      <version>3.5.2</version>
    </plugin>
    <plugin>
      <groupId>org.apache.maven.plugins</groupId>
      <artifactId>maven-jar-plugin</artifactId>
      <version>3.4.2</version>
    </plugin>
  </plugins>
</build>
```

Versions inherited through a project parent are just as explicit as versions
written in the current file. Versions inherited only from Maven's own
packaging mappings make the build depend on the Maven distribution—which
is another reason the wrapper matters.

To discover what a plugin or goal accepts, ask Maven instead of guessing
configuration element names:

```sh
./mvnw help:describe \
  -Dplugin=org.apache.maven.plugins:maven-compiler-plugin \
  -Dgoal=compile -Ddetail
```

The output includes the goal's parameters and, where one exists, the user
property accepted through `-D`.

## What a goal actually is: easing into the Mojo concept

Step back to what a "goal" even is. Every phase in the lifecycle above is
just a named checkpoint; the actual work happens in goals, and a goal's
full name always has the shape `plugin-prefix:goal-name` —
`jar:jar`, `compiler:compile`, `surefire:test`. When `mvn package` ran
back in the first example, Maven didn't have packaging logic of its own
to fall back on — it looked up which goal is bound to the `package`
phase for a `jar`-packaged project (`jar:jar`, as it turns out) and ran
exactly that.

So what runs when a goal runs? Not a shell command, not a script written
in some Maven-specific DSL — a plugin is just an ordinary jar file, and a
goal is one specific Java class inside it. Maven's name for that class is
a **Mojo**, a deliberately silly backronym for "**M**aven plain **O**ld
**J**ava **O**bject," riffing on Java's well-known POJO. The name is
doing real work as a hint: a Mojo isn't a special build-script format or
a "task" with framework machinery wired through it — it's a class like
any other, plus one addition, an annotation telling Maven "this class
implements a goal, and here's its name":

```java
@Mojo(name = "jar", defaultPhase = LifecyclePhase.PACKAGE)
public class JarMojo extends AbstractMojo {

    @Parameter(defaultValue = "${project.build.directory}", required = true)
    private File outputDirectory;

    @Parameter(property = "jar.finalName",
               defaultValue = "${project.build.finalName}")
    private String finalName;

    public void execute() throws MojoExecutionException { /* ... */ }
}
```

When Maven runs this goal, it doesn't call a constructor and pass
arguments — it instantiates the Mojo through Maven's own dependency
injection container (Plexus, or Sisu in modern Maven) and then sets each
`@Parameter` field (or calls an annotated setter) from the merged plugin
`<configuration>` and the parameter's annotation metadata. A
`defaultValue` such as `${project.build.directory}` can pull from the
effective project model above. A parameter that declares
`property = "jar.finalName"` can also be set with
`-Djar.finalName=...`; parameters without a user-property mapping are not
automatically addressable by inventing a similarly named `-D` switch.

This is why POM `<configuration>` blocks map so cleanly onto plugin
documentation pages: that documentation is generated from the same
parameter metadata. It is also why configuration errors vary in quality.
An unknown element may be ignored by a permissive plugin or binding path,
while a value that cannot be converted to the parameter's Java type can
fail immediately. `help:describe -Ddetail` and the plugin's own parameter
reference are more reliable than assuming every field supports the same
POM/property/default precedence.

This also means a Mojo, once running, is ordinary Java with access to the
whole JVM: nothing stops a plugin from reading environment variables,
making network calls, or touching files it never declared — Maven's
lifecycle model constrains *when* code runs, not *what* it's allowed to
touch, unlike Bazel's sandboxed actions.

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

## How the classpath itself gets built and used

A "classpath" isn't a Maven concept at all — it's a `java` command-line
argument (`-cp jar1:jar2:jar3`, colon-separated on Unix,
semicolon-separated on Windows) or, for an executable jar, a `Class-Path`
line in `META-INF/MANIFEST.MF`. Maven's scope resolution above produces,
for each phase that needs one, an ordered list of jar paths, which
`mvn dependency:build-classpath` will print directly:

```sh
mvn dependency:build-classpath -Dmdep.outputFile=cp.txt
cat cp.txt
```

```
/home/user/.m2/repository/org/junit/jupiter/junit-jupiter/5.11.4/junit-jupiter-5.11.4.jar:/home/user/.m2/repository/org/junit/jupiter/junit-jupiter-api/5.11.4/junit-jupiter-api-5.11.4.jar:/home/user/.m2/repository/org/junit/jupiter/junit-jupiter-engine/5.11.4/junit-jupiter-engine-5.11.4.jar:/home/user/.m2/repository/org/opentest4j/opentest4j/1.3.0/opentest4j-1.3.0.jar
```

One flat, colon-separated line — this is the literal string that ends up
after `java`'s `-cp` flag, junit's own transitive dependencies
(`opentest4j`, the jupiter engine) resolved and appended right alongside
the dependency you actually declared.

The `maven-surefire-plugin` (running `mvn test`) and the `exec-maven-plugin`
(running `mvn exec:java`) each build their own `-cp` argument this way
before forking or reflectively invoking the JVM — this is worth knowing
because it means "why does `mvn exec:java` see a different set of classes
than my IDE's run configuration" is almost always two different classpath
constructions disagreeing, not a Maven bug.

The JVM's classloader has no concept of versions or Maven coordinates —
it's a flat, ordered list, and for any given class name it loads whichever
jar on the list contains that class *first*, then never looks further,
even if a later jar on the same classpath has a class by the same name.
This is the actual mechanism "nearest wins" (below) exists to keep sane:
Maven's dependency resolution decides *which single jar* referencing a
given coordinate ends up on the classpath at all, but even after that,
two *different* libraries that happen to define classes under the same
package and class name (a "split package," or a shaded, unrelabeled
copy of a common dependency) will silently shadow one another based on
classpath order alone — no error, no warning, just whichever class
happened to load first winning, and the other library's calls into that
package getting the wrong bytecode.

An executable jar's manifest `Class-Path` entry only supports space-separated
*relative paths to other files*, not wildcards or directory globs, and
tools resolving it must know where the jar itself lives on disk to resolve
those relative paths — impractical once a project has more than a handful
of dependencies. This is the actual reason **shaded** (`maven-shade-plugin`)
and **fat/uber** jars exist: rather than shipping a jar plus a `Class-Path`
manifest pointing at forty other jars that all have to ship alongside it in
the exact right relative locations, the shade plugin unpacks every
dependency's `.class` files and merges them into one single jar with one
flat classpath entry (`java -jar app.jar`, no `-cp` needed at all) — at the
cost of reintroducing the split-package shadowing problem *inside* a
single artifact, which is why the shade plugin ships relocation and
merge-strategy configuration specifically to catch and resolve those
collisions at build time instead of at runtime.

This flat-namespace shadowing problem — two jars, same package and class
name, whichever loads first silently wins — isn't something Maven itself
ever fixed. Java 9 shipped a fix at the language and JVM level instead,
which is worth a proper detour before moving on, since it's usually
introduced in a single confusing sentence everywhere else too.

## A brief detour: what the Java module system actually is

Java shipped for two decades with exactly one way to organize code above
a single class: packages, loaded off the flat classpath just described.
That flatness is *why* split-package shadowing is possible at all —
nothing stops two unrelated jars from both defining a class named
`com.example.util.Helper`, and nothing enforces that a package's
"internal" classes stay internal. `public` was the only visibility Java
offered above the class level, and it meant public to the entire
classpath, not just to the callers a library author actually intended.

**JPMS — the Java Platform Module System, added in Java 9 (JEP 261)** —
is Java's answer. A `module-info.java` file at the root of a source tree
names the module and declares, explicitly, what it needs from other
modules and what it's willing to share:

```java
// module-info.java
module com.example.myapp {
    requires java.sql;
    requires org.apache.commons.lang3;

    exports com.example.myapp.api;
    // com.example.myapp.internal is NOT exported —
    // and now that's an enforced fact, not a naming convention
}
```

`requires` lists the other modules this one depends on — the same
dependency edges a POM already declares, just visible to the JVM itself
now, not only to Maven. `exports` is the genuinely new idea: only the
packages listed there are visible to other modules, at compile time *and*
at runtime. Every other package in the module is invisible outside it,
regardless of whether its classes are declared `public` — that's real
enforcement, not documentation. Code in another named module cannot
compile directly against a non-exported package. Deep reflective access is
a related but separate rule: packages can be opened with `opens` (or a
targeted `opens ... to ...`), and denied reflective access commonly surfaces
as `InaccessibleObjectException`. Exports govern ordinary access; opens
govern deep reflection.

Modules are consumed through a mechanism that sits *alongside* the
classpath rather than replacing it: the **module path**
(`--module-path`, distinct from `-cp`). A JVM launched with
`--module-path` resolves `requires`/`exports` relationships between named
modules instead of flattening everything into one namespace — which is
precisely what closes the split-package hole described above: two
modules exporting the same package name is a hard error at startup, not
a silent, load-order-dependent shadow.

None of this is retroactive, which is the detail that matters most for a
Maven build. A jar with no `module-info.class` — the overwhelming
many jars published before Java 9 — can still work on the module path as
an **automatic module**. Java derives or reads its module name, the
automatic module reads all other resolved modules, and all of its packages
are treated as exported and open. That preserves compatibility, but the
automatic module itself gains little of JPMS's encapsulation.

A non-modular jar can instead remain on the plain classpath as part of the
JVM's **unnamed module**, which behaves much like pre-Java-9 Java: a flat
namespace with no package exports. Hybrid applications are possible, and
named modules still receive encapsulation benefits even when some edges
lead to automatic modules. What weakens is the end-to-end guarantee: any
code left in automatic or unnamed modules retains broader visibility and
the old classpath's collision risks. This is why a Maven project can be
partly modularized rather than waiting for every third-party dependency,
but also why many applications continue to ship entirely on the classpath.

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

```sh
mvn dependency:tree -Dverbose
```

```
[INFO] com.example:my-app:jar:1.0.0
[INFO] +- com.example:library-a:jar:2.0.0:compile
[INFO] |  \- org.apache.commons:commons-lang3:jar:3.9:compile
[INFO] \- com.example:library-b:jar:1.5.0:compile
[INFO]    \- (org.apache.commons:commons-lang3:jar:3.12.0:compile - omitted for conflict with 3.9)
```

`library-a` is declared first and sits at the same depth as `library-b`,
so its `commons-lang3:3.9` is the one that's nearer by declaration order
and wins; `3.12.0` shows up in parentheses specifically *because* of
`-Dverbose` — without it, that line simply wouldn't be there, and the
tree would look like `3.9` was the only version anyone ever asked for.
The `-Dverbose` flag is what surfaces the versions that *lost* the
resolution and why, not just the winner — without it the tree only shows
what got resolved, which is the wrong direction to debug from when a
runtime `NoSuchMethodError` says some class changed shape between versions.

## `dependencyManagement` chooses versions; `dependencies` chooses artifacts

These two blocks look deceptively similar. An entry in `<dependencies>`
actually adds an artifact to the project. An entry in
`<dependencyManagement>` supplies defaults—most importantly a version—if
the current project or a child later adds that artifact:

```xml
<dependencyManagement>
  <dependencies>
    <dependency>
      <groupId>org.junit</groupId>
      <artifactId>junit-bom</artifactId>
      <version>5.11.4</version>
      <type>pom</type>
      <scope>import</scope>
    </dependency>
  </dependencies>
</dependencyManagement>

<dependencies>
  <dependency>
    <groupId>org.junit.jupiter</groupId>
    <artifactId>junit-jupiter</artifactId>
    <scope>test</scope>
  </dependency>
</dependencies>
```

The first dependency is a **BOM**—a Bill of Materials, packaged as a POM.
`type=pom` plus `scope=import` imports its managed dependency versions into
this POM's dependency-management section. The second declaration actually
adds JUnit and obtains its version from the BOM. BOMs let a framework
publisher test and release a coherent set of component versions rather than
making every consumer align them independently.

A direct managed version also participates in dependency mediation. A
parent can pin `commons-lang3` to an approved version even when it enters
only transitively. That gives a build control over nearest-wins, but it
also transfers responsibility: forcing a version doesn't prove every
library in the graph is compatible with it.

`<pluginManagement>` is the build-plugin counterpart. It supplies versions
and configuration when a plugin is used by a child, but—except for goals
already supplied by a packaging lifecycle binding—it does not activate the
plugin merely by mentioning it. A parent commonly keeps policy in both
management blocks while child modules state which dependencies and plugins
they actually need.

## Exclusions and `optional` dependencies change graph edges

If one transitive dependency is unwanted, exclude it at the edge through
which it arrives:

```xml
<dependency>
  <groupId>com.example</groupId>
  <artifactId>library-a</artifactId>
  <version>2.0.0</version>
  <exclusions>
    <exclusion>
      <groupId>commons-logging</groupId>
      <artifactId>commons-logging</artifactId>
    </exclusion>
  </exclusions>
</dependency>
```

An exclusion is not a global ban. If another path brings the same artifact
in, it remains in the graph. Use `dependency:tree` after adding one and
state the replacement dependency explicitly when the application still
needs that API.

A library author can instead mark one of its dependencies `<optional>true>`.
The dependency remains available while compiling and testing that library,
but consumers do not inherit it transitively. Optional is appropriate for
features that only some consumers enable; those consumers declare the
corresponding dependency themselves. It is not a way to hide a dependency
required by the library's normal runtime path.

## The local repository and how Maven decides to hit the network

`~/.m2/repository` isn't just a download folder — it's a cache keyed by
GAV coordinate, laid out as
`groupId/artifactId/version/artifactId-version.jar` (plus a `.sha1`
checksum alongside it, verified on download). For an ordinary release
version, once it's there and checksum-verified, Maven never asks the
remote repository about it again — release coordinates are treated as
immutable forever, which is the entire reason Maven Central rejects
re-uploads of an already-published version rather than allowing
overwrites.

`-SNAPSHOT` versions are the deliberate exception. A dependency on
`1.0.0-SNAPSHOT` doesn't resolve to one artifact — the remote repository
keeps a `maven-metadata.xml` in that version's directory listing the
actual latest build, timestamped and numbered
(`1.0.0-20260114.093000-7`), and Maven consults it to find the real
artifact to fetch. How often it re-checks is governed by an update
policy — `daily` by default — which is why a snapshot dependency that
just got published five minutes ago sometimes doesn't show up in a build
until `mvn install -U` (`-U` forces the metadata re-check regardless of
policy) is used to force it. It's the same tension the "Snapshot versions
resolving differently across machines" note below describes, just from
the mechanism's side: the local cache is what makes Maven fast, and
snapshot metadata is the deliberate crack in that cache that makes
snapshots useful for active development.

## settings.xml, mirrors, and why `.m2` doesn't always mean Central

Maven Central being the default remote repository (inherited from the
Super POM, as noted above) doesn't mean every `.m2` cache actually talks
to it. `~/.m2/settings.xml` (or an `.mvn/settings.xml` bundled with the
project) can declare a **mirror** that intercepts requests for one or
more repository IDs and redirects them elsewhere entirely:

```xml
<settings>
  <mirrors>
    <mirror>
      <id>internal-nexus</id>
      <mirrorOf>*</mirrorOf>
      <url>https://nexus.internal.example.com/repository/maven-public/</url>
    </mirror>
  </mirrors>
</settings>
```

`<mirrorOf>*</mirrorOf>` means *every* repository lookup, including
Central itself, gets rewritten to hit the internal server instead — a
common corporate setup, since it lets one proxy cache Central, enforce an
artifact allowlist, and host internally-published artifacts through the
same URL. The practical consequence: the exact same `pom.xml`, on two
machines with different `settings.xml` files, can resolve the identical
GAV coordinate from two completely different servers, which is invisible
from the POM alone — `mvn help:effective-settings` is the settings-side
counterpart to `mvn help:effective-pom`, and it's the first thing to run
when a dependency resolves differently on a CI runner than it does
locally.

## Offline mode and recovering a broken cache entry

`mvn -o` (offline mode) refuses to contact any remote repository at
all — every dependency has to already be sitting in `~/.m2/repository`,
checksum-verified, or the build fails immediately instead of hanging on a
network call. This is mainly useful for two opposite situations: proving
a build is fully reproducible from what's already cached, and working
somewhere genuinely disconnected, where a fast, clear failure beats a
slow timeout against an unreachable server.

A cache entry can also end up **corrupted** — a download interrupted
partway through, or (rarer, but it happens) a `.jar` that downloaded
successfully but doesn't match its `.sha1`. Maven's checksum verification
catches the mismatch, but the fix isn't always obvious from the error
alone, because by default Maven doesn't re-download something it already
believes it has:

```
mvn dependency:purge-local-repository -DmanualInclude=com.example:my-lib
```

This deletes the cached artifact (and, depending on flags, its resolution
metadata) so the next build re-fetches it clean, rather than continuing
to serve the broken copy. The blunter version — manually deleting the
specific `groupId/artifactId/version` directory under `~/.m2/repository`,
or the whole cache when the scope of the corruption is unclear — works
for the same reason `rm -rf target/` works for a stale build: the cache
is disposable and rebuildable by definition, so when in doubt about what
exactly went stale, clearing more of it is a safe, if slower, way to get
back to a known-good state.

## The reactor: multi-module builds

Two Maven relationships are often placed in the same root `pom.xml`, but
they are not the same:

- **aggregation** is the root's `<modules>` list and tells the reactor which
  projects participate in one invocation;
- **inheritance** is a child's `<parent>` element and tells model building
  which properties, management, and plugin configuration flow into that
  child.

An aggregator with `<modules>` turns a directory of related projects into
one build:

```xml
<project>
  <groupId>com.example</groupId>
  <artifactId>my-app-parent</artifactId>
  <version>1.0.0-SNAPSHOT</version>
  <packaging>pom</packaging>
  <modules>
    <module>core</module>
    <module>api</module>
    <module>web</module>
  </modules>
</project>
```

Each child can inherit from it:

```xml
<parent>
  <groupId>com.example</groupId>
  <artifactId>my-app-parent</artifactId>
  <version>1.0.0-SNAPSHOT</version>
  <relativePath>../pom.xml</relativePath>
</parent>
<artifactId>core</artifactId>
```

But neither relationship requires the other. A corporate parent POM can be
published and inherited by hundreds of unrelated repositories without
aggregating them. Conversely, an aggregator can build modules whose parent
is somewhere else. Calling every root POM “the parent” works until a build
uses those relationships independently.

Running `mvn install` from the parent invokes the **reactor**, which reads
every module's POM, builds a dependency graph between the modules themselves
(if `web` depends on `api`, `api` must build first), and topologically sorts
the build order — this is Make's dependency-graph idea again, just applied
at the level of whole modules instead of individual files. `mvn install
-pl web -am` builds only `web` and everything it transitively depends on
(`-am` = "also make") — the reactor equivalent of Make only rebuilding the
stale part of the graph, useful once a multi-module build gets big enough
that a full `mvn install` takes minutes instead of seconds.

Two companion switches are useful when repairing a large reactor:

```sh
./mvnw verify -pl web -am   # selected project plus required upstream modules
./mvnw verify -pl core -amd # selected project plus downstream dependents
./mvnw verify -rf :api      # resume from artifactId api after a failure
```

## Parallel reactor builds, and why old plugins break under them

`mvn install -T 4` (or `-T 1C`, one thread per CPU core) tells the reactor
to build independent modules concurrently instead of one at a time,
respecting the same module dependency graph used for ordering — modules
with no path between them in that graph run on separate threads, modules
on either side of a dependency edge still run in order.

This exposes a class of bug that a serial build never triggers: a Mojo
written years before parallel builds existed, holding mutable state in a
`static` field (a shared cache, a counter, a "have I already logged this
warning" flag) that was never a problem when only one module's build ever
touched it at a time. Under `-T`, two modules' Mojo instances can now
execute that same static field concurrently, and the result is a build
that's flaky specifically under `-T` and specifically nondeterministic —
different modules racing on different runs — which is exactly the
signature that points back to shared mutable plugin state rather than
anything wrong with your own POM. Plugins declare themselves safe for
this via `@Mojo(threadSafe = true)`; one that doesn't declare it is
assumed unsafe and Maven will warn rather than silently parallelize it.

## Profiles: conditionally different effective POMs

A `<profile>` is a fragment of POM (dependencies, plugins, properties)
that only merges into the effective POM if its activation condition
holds — a system property, an OS, a JDK version range, a file's
presence or absence, or `<activeByDefault>true</activeByDefault>`:

```xml
<profiles>
  <profile>
    <id>integration-tests</id>
    <activation>
      <property><name>env.CI</name></property>
    </activation>
    <build>
      <plugins>
        <plugin><!-- extra verification, only under CI --></plugin>
      </plugins>
    </build>
  </profile>
</profiles>
```

The sharp edge: `activeByDefault` only means "active when *nothing else*
activates a profile." The moment any profile is turned on explicitly —
`mvn install -Pintegration-tests` — every `activeByDefault` profile
stops applying, silently, even ones with no relationship to the one you
named. A build that behaves differently the instant someone adds `-P` for
an unrelated reason is almost always this rule, not a bug in whichever
profile was just activated. `mvn help:active-profiles` shows what's
actually live for a given invocation, which is the direct way to check
rather than inferring it from `<activation>` blocks by eye.

Profiles are best used for genuinely environmental variation, such as
enabling an expensive deployment check or selecting platform-specific
native tooling. Using profiles to manufacture “dev,” “test,” and “prod”
application artifacts from the same source tends to make the artifact's
meaning depend on an invisible command-line flag. Prefer building one
artifact and supplying runtime configuration at deployment.

## Compiler release, the running JDK, and toolchains are separate knobs

`<maven.compiler.release>17</maven.compiler.release>` tells `javac` which
Java language level and standard-library API the output may use. It does
not locate a JDK 17 installation, and it does not stop Maven itself from
running under some other supported JDK. Check both Maven and Java with:

```sh
./mvnw --version
```

When different plugins must consistently use a particular JDK, configure
the Maven Toolchains plugin and keep machine-specific installation paths
outside the project POM. A user-level `~/.m2/toolchains.xml` can describe
available JDKs:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<toolchains>
  <toolchain>
    <type>jdk</type>
    <provides>
      <version>21</version>
      <vendor>temurin</vendor>
    </provides>
    <configuration>
      <jdkHome>/opt/jdks/temurin-21</jdkHome>
    </configuration>
  </toolchain>
</toolchains>
```

The project then requests capabilities rather than embedding that path:

```xml
<plugin>
  <groupId>org.apache.maven.plugins</groupId>
  <artifactId>maven-toolchains-plugin</artifactId>
  <version>3.2.0</version>
  <executions>
    <execution>
      <goals><goal>select-jdk-toolchain</goal></goals>
    </execution>
  </executions>
  <configuration>
    <version>21</version>
    <vendor>temurin</vendor>
  </configuration>
</plugin>
```

Compiler, Surefire, Javadoc, and other toolchain-aware plugins can then use
the selected JDK even if the JVM hosting Maven is different. The wrapper
pins Maven; toolchains make external build tools explicit; neither removes
the need to document which JDK is allowed to launch Maven itself.

## Publishing to Maven Central: more gates than a plain `mvn deploy`

For an internal Nexus or Artifactory repository, `mvn deploy` normally
uploads to the repository declared by `<distributionManagement>`:

```xml
<distributionManagement>
  <repository>
    <id>internal-releases</id>
    <url>https://repo.example.com/repository/maven-releases/</url>
  </repository>
</distributionManagement>
```

Maven Central's current Publisher Portal is not that kind of repository
URL. Its Maven extension intercepts `deploy`, assembles a publishing
bundle, and uploads the bundle through the Portal API:

```xml
<plugin>
  <groupId>org.sonatype.central</groupId>
  <artifactId>central-publishing-maven-plugin</artifactId>
  <version>0.9.0</version>
  <extensions>true</extensions>
  <configuration>
    <publishingServerId>central</publishingServerId>
  </configuration>
</plugin>
```

The matching `<server>` in `~/.m2/settings.xml` holds a Portal user token;
credentials do not belong in the project POM. With the extension active,
`./mvnw deploy` creates and uploads a deployment for Portal validation.
Depending on plugin configuration, a validated deployment is either
published automatically or released through the Portal afterward.

Central is a public registry Maven builds trust by default, so accepting
the upload requires more than producing a usable local JAR.

**Signing.** Central rejects unsigned uploads outright. `maven-gpg-plugin`,
bound to the `verify` phase, signs every artifact (`.jar`, `-sources.jar`,
`-javadoc.jar`, and the POM itself) with a GPG key whose public half is
published to a keyserver, producing a detached `.asc` signature alongside
each file:

```xml
<plugin>
  <groupId>org.apache.maven.plugins</groupId>
  <artifactId>maven-gpg-plugin</artifactId>
  <executions>
    <execution>
      <id>sign-artifacts</id>
      <phase>verify</phase>
      <goals><goal>sign</goal></goals>
    </execution>
  </executions>
</plugin>
```

Anyone resolving the published artifact later can verify the `.asc`
against the public key. The signature files are a hard requirement for
the upload bundle to be accepted.

**POM completeness.** A POM that's perfectly sufficient for `mvn package`
locally — `groupId`/`artifactId`/`version`/`packaging` and nothing else —
fails Central's validation, because Central requires `<name>`,
`<description>`, `<url>`, a `<license>`, an `<scm>` block, and at least
one `<developer>` entry. None of these affect whether the build itself
succeeds, which is exactly why they're easy to forget until the *upload*
fails with a validation error on an artifact that built, tested, and
`install`ed into the local `.m2` cache without a single complaint.

**Sources and javadoc jars.** Central requires a `-sources.jar` and a
`-javadoc.jar` alongside the main artifact, produced by binding
`maven-source-plugin:jar-no-fork` and `maven-javadoc-plugin:jar` to the
`package` phase — an ordinary local build skips both by default, since
neither is needed to compile or test the project, so this is another gate
that's invisible until publish time.

**The upload path changed.** For years, publishing meant an account on
Sonatype's OSSRH Nexus, staging an upload via
`nexus-staging-maven-plugin`, then “closing” and “releasing” that staging
repository. The Central Publishing Portal replaced that workflow with the
token-authenticated bundle upload shown above. The validation gates remain
recognizable—signatures, complete POM metadata, sources, and Javadoc—but
old OSSRH guides name different plugins, endpoints, and credentials. Do
not combine half of each setup.

## Reproducibility is more than pinning dependency versions

A release dependency with an exact GAV is immutable by repository policy,
but that alone does not make two builds byte-for-byte equal. The result can
also depend on:

- the Maven distribution and JDK;
- build-plugin versions and their transitive dependencies;
- active profiles, system properties, and environment variables;
- repository mirrors and the bytes they serve;
- locale, timezone, filesystem ordering, and timestamps embedded in ZIP/JAR
  entries;
- generated sources or resources that contain the current time, hostname,
  or absolute checkout path.

The wrapper, toolchains, exact release dependencies, a parent/BOM, and
pinned plugins constrain the first half of that list. For archive output,
Maven plugins increasingly honor a shared timestamp property:

```xml
<properties>
  <project.build.outputTimestamp>2026-01-01T00:00:00Z</project.build.outputTimestamp>
</properties>
```

Many projects set it to the timestamp of the source commit during a release,
so rebuilding the same commit does not stamp every JAR entry with the wall
clock time. This only helps plugins that honor the property, and it cannot
repair a generator that embeds nondeterministic content of its own.

The practical test is to build the same revision twice from clean,
independent directories and compare checksums. “Every declared dependency
has a version” is an important input audit, not proof of reproducibility.

## A debugging ladder for unfamiliar Maven failures

Start with the narrowest command that exposes the relevant merged state:

| Question | Command |
|---|---|
| Which Maven and Java am I running? | `./mvnw --version` |
| What did inheritance and profiles produce? | `./mvnw help:effective-pom -Dverbose` |
| Which settings, mirrors, and servers apply? | `./mvnw help:effective-settings` |
| Which profiles are active? | `./mvnw help:active-profiles` |
| Which dependency version won? | `./mvnw dependency:tree -Dverbose` |
| Why is an artifact present? | `./mvnw dependency:tree -Dincludes=group:artifact` |
| What parameters does a goal accept? | `./mvnw help:describe -Dplugin=g:a -Dgoal=name -Ddetail` |
| Which reactor projects and order did Maven select? | read the “Reactor Build Order” near startup |
| What exception is hidden behind the summary? | rerun with `-e` |
| What is Maven doing internally? | rerun with `-X` |

`-X` is intentionally last. Debug logs are enormous and can contain
repository URLs, environment-derived values, and configuration, so they
are harder to reason about and should be inspected before being attached
to a public issue. Effective models and focused dependency trees usually
answer the question with far less noise.

## Where people actually get burned

- **`mvn clean install` as a reflex.** The fixed lifecycle makes Maven look
  deterministic, but plugins can and do leave stale state (annotation
  processor output, generated sources) that a partial rebuild doesn't
  regenerate correctly. `clean` deletes `target/` first — it's the
  timestamp-cache-busting equivalent of `touch`-ing every file in a Make
  build, and it becomes habitual precisely because Maven's incrementality is
  weaker than Make's or Gradle's.
- **Management mistaken for activation.** Neither
  `<dependencyManagement>` nor `<pluginManagement>` generally adds the
  thing it describes. A child still declares the dependency or plugin it
  uses; management supplies the inherited version and configuration.
- **Snapshot versions resolving differently across machines.** A `-SNAPSHOT`
  dependency is mutable — Maven re-resolves it from the repository on a
  schedule rather than treating it as immutable once downloaded, which is
  exactly the property a reproducible build needs *not* to have. It's fine
  for local development against a library you're actively changing, and a
  liability the moment it survives into a release build.
- **`package` passes while integration tests never ran.** Failsafe lives
  after `package`; invoke `verify` when the build's quality gates matter.
- **The IDE and command line use different JDKs or profiles.** Compare the
  IDE's Maven runner, environment, active profiles, and user settings with
  `./mvnw --version`, `help:active-profiles`, and the effective settings
  before blaming dependency resolution.
