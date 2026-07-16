# Software Plumbers

📖 **Read it here → [vivainio.github.io/sw-plumber-book](https://vivainio.github.io/sw-plumber-book/)**

Notes on the build tools, dependency managers, and CI pipelines that move
code from a source tree to a running artifact — Make, Maven, Gradle,
MSBuild, npm, and the plumbing that connects them. The unglamorous layer
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
