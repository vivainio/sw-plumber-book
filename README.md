# Software Plumbers

📖 **Read it here → [vivainio.github.io/sw-plumber-book](https://vivainio.github.io/sw-plumber-book/)**

Notes on the plumbing underneath software: build tools, dependency
managers, and CI pipelines (Make, Maven, Gradle, MSBuild, npm), but also
the debuggers, profilers, linkers, and OS-level machinery that let you see
inside — and reason about — a running process. The unglamorous layer
everyone depends on and almost nobody enjoys reading about, explained with
real examples instead of marketing copy.

## Building locally

The site is built with [Zensical](https://zensical.org) and published to GitHub
Pages.

```sh
zensical serve          # live preview at http://localhost:8000
zensical build --clean  # static build into ./site
```

Content lives in [`docs/`](docs/); navigation is configured in
[`zensical.toml`](zensical.toml).
