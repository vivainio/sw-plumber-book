---
icon: lucide/package
---

# npm, pnpm, and Yarn

The Node ecosystem's build tool is barely a build tool at all — for most
projects it's `package.json`'s `scripts` block calling out to whatever
actually does the work (`tsc`, `webpack`, `vite`), plus a dependency
resolver bolted on. That resolver is where the real engineering is, and
it's also where npm, Yarn, and pnpm genuinely diverge from each other —
not in syntax, but in what `node_modules` looks like on disk once
installation finishes.

## package.json scripts: the smallest build tool that could work

```json
{
  "scripts": {
    "build": "tsc",
    "test": "vitest run",
    "prepublishOnly": "npm run build"
  }
}
```

`npm run build` runs `tsc`. There's no dependency graph, no incremental
build tracking, no task-skips-if-unchanged logic — every script runs in
full, every time, unless the underlying tool it calls (like `tsc
--incremental`) does its own caching. This is worth naming plainly, because
compared to Make's timestamp graph or Gradle's hashed inputs/outputs,
`npm run` is a dumber mechanism by design: a name-to-shell-command map, and
nothing more. Some npm script names are conventions the CLI itself invokes
automatically — `prepublishOnly` before `npm publish`, `postinstall` after
`npm install` completes — which is also the ecosystem's most-abused
feature: a `postinstall` script runs arbitrary code on every machine that
installs the package, with no sandboxing, which is the mechanism behind a
meaningful share of supply-chain npm incidents.

## node_modules resolution: why nesting existed and why it stopped

Node's `require()`/`import` resolution walks up the directory tree looking
for a `node_modules` folder containing the requested module — this
algorithm predates npm's dependency-tree design and constrains it. Early
npm (v2) nested dependencies literally: if `A` depends on `B` which depends
on `C`, the disk layout was `node_modules/A/node_modules/B/node_modules/C`.
This mirrored the logical dependency tree exactly and made version
conflicts impossible — nested copies never collide — at the cost of
massive duplication and, on Windows, hitting the OS path-length limit on
any moderately deep tree.

npm v3 (and Yarn, from its first release) introduced **flattening**:
hoist every package to the top level of `node_modules` when there's no
version conflict, and only nest a package where a real conflict requires
it.

```
node_modules/
├── A/
├── B/           ← hoisted, both A and top-level need it
└── C/
```

This fixed the duplication and path-length problems, but introduced
**phantom dependencies** — code can `require('B')` even if only `A`
declared a dependency on `B`, purely because flattening happened to place
`B` at the top level where Node's resolution algorithm finds it. Nothing in
`package.json` says your code depends on `B`; the import only works because
of how the *current* tree happened to flatten. Add or remove an unrelated
dependency elsewhere in the tree, change the flattening outcome, and code
that worked yesterday throws `Cannot find module` today, for a dependency
that was never declared to begin with.

## pnpm: symlinks and a content-addressable store

pnpm's answer to phantom dependencies is a completely different
`node_modules` layout. Every package version is stored once in a global,
content-addressable store (`~/.local/share/pnpm/store` or similar), and a
project's `node_modules` is built from **symlinks** into that store, with a
strict two-level structure — a real dependency, and a hidden
`.pnpm/`-namespaced tree that only exposes exactly what each package
actually declared:

```
node_modules/
├── .pnpm/
│   ├── a@1.0.0/node_modules/A -> ../../A
│   ├── b@2.0.0/node_modules/B -> ...
│   └── ...
└── A -> .pnpm/a@1.0.0/node_modules/A
```

`require('B')` from your own code fails unless `B` is an actual
`package.json` dependency, because pnpm's symlink structure only exposes
declared dependencies at the top level — the phantom-dependency bug class
is structurally impossible, not just discouraged. The shared content store
also means installing the same version of a package across ten projects on
one machine stores it on disk once, not ten times — a genuinely different
disk-usage and install-speed profile from npm or Yarn classic, and the main
reason large monorepos gravitate toward it.

## Yarn: two different tools sharing one name

"Yarn" means two things people conflate constantly. **Yarn Classic (v1)**
is a faster, deterministic-lockfile alternative to npm v2/v3-era `npm
install`, with roughly the same flattened `node_modules` layout npm
eventually adopted. **Yarn Berry (v2+)** is a different project entirely,
best known for **Plug'n'Play (PnP)** — it can skip `node_modules` on disk
altogether, generating a single `.pnp.cjs` file that maps package names
directly to locations inside zipped package archives, and hooking Node's
module resolution to consult that map instead of walking directories. This
removes the entire class of `node_modules` layout problems (phantom deps,
flattening, disk usage) at the cost of breaking any tool that assumes
`node_modules` exists on disk as real files — which, in practice, was
enough of the ecosystem for years that PnP adoption stayed a minority
choice long after it shipped.

## Lockfiles: the same job, incompatible formats

`package-lock.json` (npm), `yarn.lock` (Yarn), and `pnpm-lock.yaml` (pnpm)
all solve the identical problem: `package.json` version ranges
(`^1.2.0`) describe what's *acceptable*, not what's *installed*, so
without a lockfile, two installs a week apart can silently resolve
different actual versions. All three lockfiles pin exact resolved versions
(and, for registries that support it, content hashes) so a fresh `install`
reproduces the same tree bit-for-bit. None of the three formats are
interchangeable, and none of the tools reliably reads another's lockfile —
switching package managers in an existing project means regenerating the
lockfile from scratch and treating any dependency version *drift* that
surfaces as expected, not a bug in the new tool.

## Workspaces: one lockfile, many packages

A monorepo with multiple `package.json` files (a common shape for a
frontend app plus a shared component library) uses **workspaces** — a
top-level `package.json` listing member paths:

```json
{
  "private": true,
  "workspaces": ["packages/*"]
}
```

All three package managers support the concept, with the same practical
effect: one lockfile for the whole repo, shared dependencies hoisted to the
root `node_modules` once instead of duplicated per package, and a local
package (`packages/ui`) resolvable by name from a sibling package
(`packages/app`) via a symlink the package manager creates automatically —
no publishing to a registry required for one workspace package to depend on
another. This is npm's version of Maven's multi-module reactor: one install
operation, one graph, instead of independently installing each package's
dependencies in isolation.
