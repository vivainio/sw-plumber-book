---
icon: lucide/layers-3
---

# Virtual Environments: Isolation Without a Container

A Python virtual environment looks, from the outside, like a self-contained
Python install: its own `bin/python`, its own `site-packages`, its own
`pip`. It isn't. There is exactly one CPython interpreter build involved —
the venv's `python` is either a symlink to the system interpreter or a tiny
copy of its launcher stub — and "isolation" is really just that stub
starting up, noticing where it lives, and rewriting `sys.path` before your
code ever runs. No namespaces, no filesystem jail, none of the machinery
the [containers](../containers/index.md) chapter covers. A venv is a
directory layout and a handful of environment variables the interpreter
knows to look for — which is exactly why it's cheap to create thousands of
them and why, when something goes wrong, the fix is almost always "look at
`sys.path`."

## What `python -m venv` actually creates

```
$ python -m venv myenv
$ find myenv -maxdepth 2
myenv/pyvenv.cfg
myenv/bin/python -> /usr/bin/python3.11
myenv/bin/python3 -> python
myenv/bin/pip
myenv/bin/activate
myenv/lib/python3.11/site-packages/
```

`bin/python` is a **symlink** back to the interpreter that created it (on
Windows, `venv` copies `python.exe` instead, since Windows symlinks need
elevated privileges and DLL loading is picky about the executable's own
directory). `lib/python3.11/site-packages/` starts empty except for
whatever `pip`, `setuptools`, and `wheel` the venv was seeded with. Nothing
here contains a second copy of the standard library — the venv borrows the
base interpreter's `lib/python3.11/` wholesale and only diverges for
third-party packages. That's the entire trick: one real interpreter, many
cheap, disposable `site-packages` directories pointed at it.

## `pyvenv.cfg`: the file that makes a symlink into an environment

```ini
home = /usr/bin
include-system-site-packages = false
version = 3.11.4
executable = /usr/bin/python3.11
command = /usr/bin/python3.11 -m venv /home/user/myenv
```

On startup, CPython's C-level `getpath` initialization walks up from
`sys.executable` looking for a `pyvenv.cfg` in the same directory or its
parent. Finding one is what flips the interpreter into "I'm running inside
a venv" mode — it's not detected from an environment variable set at
launch, it's discovered from disk, every single time the interpreter
starts. `home` tells it which real installation's standard library to fall
back to; `include-system-site-packages` decides whether the base
interpreter's `site-packages` gets appended to `sys.path` as well (`false`
by default, which is the whole point — a venv with this set to `true` is
only as isolated as the system environment underneath it, a footgun some
CI configs enable by accident to save install time and then get bitten by
a version pinned differently than what the venv wants).

## Activation is three environment variables, not a mode

```sh
$ which python
/usr/bin/python
$ source myenv/bin/activate
(myenv) $ which python
/home/user/myenv/bin/python
$ echo $VIRTUAL_ENV
/home/user/myenv
$ echo $PATH | tr ':' '\n' | head -1
/home/user/myenv/bin
```

`activate` is a shell script, not a Python mechanism — it prepends
`myenv/bin` to `$PATH`, sets `$VIRTUAL_ENV`, tweaks `$PS1` for the prompt,
and stashes the old `$PATH` so `deactivate` can restore it. That's all.
Nothing about it talks to the interpreter, and nothing about it is
required: running `myenv/bin/python script.py` directly, with no
activation at all, produces the identical isolated `sys.path` — because
the isolation lives in `pyvenv.cfg` discovery at interpreter startup, not
in the shell state. This is also why "which venv am I in?" bugs are so
common in scripts and CI: activating in one shell, then launching a
subprocess or a new SSH session, means `$PATH` doesn't carry over, and the
subprocess silently runs the system `python` — no error, just the wrong
interpreter with none of the packages you expected.

## `sys.path` construction: where imports actually get resolved

Inside an activated (or directly invoked) venv interpreter, `import`
resolution walks `sys.path` in order:

```python
>>> import sys; sys.path
['', '/usr/lib/python3.11',                     # stdlib, from `home`
 '/usr/lib/python3.11/lib-dynload',
 '/home/user/myenv/lib/python3.11/site-packages']  # venv's own packages
```

The venv's `site-packages` is appended *last*, which means a package
installed there shadows nothing in the standard library by ordering alone
— but if you `pip install requests` and also happen to have a stray
`requests.py` in your current working directory, the empty string entry
(cwd) wins, a classic source of "why is my import grabbing the wrong
file" that has nothing to do with the venv at all. `site-packages` itself
isn't scanned recursively; every package under it needs either a real
directory with an `__init__.py`/namespace-package marker, or a `.pth` file
— a plaintext file of extra paths that Python's `site` module reads at
startup and appends to `sys.path` verbatim, which is how editable installs
(`pip install -e .`) work: no copying, just a `.pth` entry (or, in modern
`pip`, a small `__editable___mypkg_finder.py` import hook) pointing back
at your source checkout.

## venv vs virtualenv: stdlib module vs third-party tool

`python -m venv` is the standard-library implementation, added in Python
3.3 (PEP 405) precisely because the ecosystem had converged on wanting
this and didn't want it to depend on a separately-versioned PyPI package.
**virtualenv**, the older third-party tool `venv` was modeled on, still
exists and still gets used, mainly because it does a few things `venv`
doesn't: it can create environments against a *different* Python version
than the one running it (given a discoverable interpreter), it seeds new
environments from a local wheel cache instead of downloading `pip` fresh
each time, and historically it supported far older Python versions that
had no `venv` module at all. Both produce the same `pyvenv.cfg`-plus-symlink
shape, and an interpreter doesn't care which tool made the environment it's
starting in — the two are interchangeable at the point of use, and the
choice between them is almost entirely about creation-time convenience.

## `PYTHONHOME` and `PYTHONPATH`: the two variables that can break a venv from outside

`pyvenv.cfg`'s `home` key is a venv-local answer to the same question
`PYTHONHOME` answers globally: where does the standard library actually
live? If `PYTHONHOME` is set in the environment, it **overrides** what a
venv's `pyvenv.cfg` would otherwise select, which is precisely the
mechanism [PyInstaller](../containers/index.md)-style bundlers and
embedded-Python applications use to point a bundled interpreter at their
own private stdlib copy — and precisely why leaving a stray `PYTHONHOME`
set in a shell profile (often left behind by one of those bundlers) causes
every venv activated afterward to import a mismatched standard library out
from under it, with import errors that look nothing like an environment
problem. `PYTHONPATH`, unlike `PYTHONHOME`, doesn't override anything —
it's *prepended* to `sys.path` ahead of the venv's own `site-packages`,
so a leftover `PYTHONPATH=/some/old/project` from a previous session can
silently shadow a package the venv installed, with the venv itself
completely uninvolved in explaining why.

## PEP 668: `externally-managed-environment` and why `pip` started refusing

```
$ pip install requests
error: externally-managed-environment

× This environment is externally managed
╰─> To install Python packages system-wide, try apt install
    python3-xyz, where xyz is the package you are trying to
    install.
```

Before PEP 668, `pip install` against a distro-packaged Python happily
wrote straight into `/usr/lib/python3/dist-packages` — the same directory
the OS package manager (`apt`, `dnf`) manages — and a `pip`-installed
version could silently overwrite or conflict with a package the distro
depends on for its own tooling. PEP 668 has distros drop an
`EXTERNALLY-MANAGED` marker file next to the system `site-packages`; `pip`
checks for it and refuses to install unless the target is a venv (or
`--break-system-packages` is passed, deliberately worded to make the
override feel like what it is). The fix isn't a `pip` flag — it's using a
venv for anything that isn't the OS's own tooling, which is precisely the
isolation this whole chapter is about: the marker exists because too much
of the ecosystem was treating the system interpreter as a scratch space.

## conda: not a `sys.path` trick, and why that changes what it can do

Everything above assumes one interpreter build with `sys.path` doing the
isolation work. **conda** environments don't work that way — `conda create
-n myenv python=3.11` installs a **complete, independent copy** of the
CPython interpreter (and, unlike venv/virtualenv, potentially a different
version than whatever created the environment) into
`envs/myenv/bin/python`, no symlink back to a shared base. That's a
heavier unit of isolation, but it buys conda something `pip` fundamentally
can't offer: its packages aren't limited to Python wheels, so a `conda
install` can pull in a specific version of a shared C library — CUDA
toolkits, MKL, a particular `libgdal` — with conda's own dependency
resolver reconciling *native* library versions across the whole
environment, not just Python-level `Requires-Dist` constraints the way
the [dependency resolution](../dependency-resolution/index.md) chapter
describes for wheels. This is why swapping a `conda` scientific-computing
environment for a `venv` isn't a drop-in change — the wheels alone don't
carry the non-Python native dependencies conda was silently managing
underneath them.

## uv: same `pyvenv.cfg` contract, a faster path to filling it

**uv** creates venvs with the identical `pyvenv.cfg`/symlink shape
`python -m venv` produces — any tool that understands a stdlib venv
understands a `uv venv` — but populates `site-packages` differently. Where
`pip` downloads and unpacks a wheel per install, uv keeps a global,
content-addressed package cache once per machine and **hardlinks** (or, on
filesystems that don't support that, copies) files from the cache into
each venv's `site-packages`, so installing a package already cached from
another project costs a handful of filesystem operations rather than a
download and unzip. It's the same tactic package managers use elsewhere —
pnpm's content-addressed store does the equivalent for `node_modules`,
covered in the [npm chapter](../npm/index.md) — applied to a part of the
Python toolchain that had, until uv, mostly ignored it. The isolation
guarantee is unaffected either way: hardlinked or copied, each venv's
`site-packages` only contains what was explicitly installed into it.

## Where people actually get burned

- **A venv silently pointing at a deleted interpreter.** `bin/python` is a
  symlink to the system interpreter's path, not a copy of it. Upgrading or
  removing that system Python (a distro upgrade, `brew upgrade python`)
  leaves every venv created from it with a dangling symlink — `python:
  command not found` or, worse, a *different* interpreter now living at
  that same path, one the venv's `pyvenv.cfg` never accounted for.
- **`include-system-site-packages = true` reintroducing exactly the
  conflict a venv exists to prevent.** Set once for convenience (usually
  to avoid reinstalling a large package like `numpy`), it means a system
  package upgrade can change what an "isolated" venv resolves to import,
  with no record of that dependency in the venv's own install history.
- **Copying a venv directory instead of recreating it.** The symlink in
  `bin/python`, the absolute paths baked into `bin/activate` and the
  `.pth`/shebang lines under `bin/`, and `pyvenv.cfg`'s `home` key are all
  written for one specific filesystem location. Move or `cp -r` the
  directory elsewhere and activation, `pip`, and every console-script
  shebang keep pointing at the old path — venvs are meant to be recreated
  from a lockfile or `requirements.txt`, never relocated.
- **A leftover `PYTHONPATH` or `PYTHONHOME` from one project's `.env`
  bleeding into an unrelated shell session.** Because both are read at
  interpreter startup regardless of which venv is active, they override
  or shadow venv isolation invisibly — the venv itself shows no sign
  anything is wrong, since from its own `pyvenv.cfg`'s perspective nothing
  changed.
- **Trusting `pip list` inside an accidentally-wrong interpreter.** Running
  `pip install` after `sudo` (which drops `$PATH` customizations and often
  the venv along with them) or inside a subprocess that didn't inherit an
  activated shell's `$PATH` installs into whatever `pip` resolves to
  globally — usually the system Python `pyvenv.cfg` never touches — and
  the failure only surfaces later, as a `ModuleNotFoundError` at runtime
  in the environment that was actually meant to have the package.
