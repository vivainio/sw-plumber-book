---
icon: lucide/box
---

# Containers

A container is not a lightweight virtual machine, even though that's the
mental model most people start with. There's no hypervisor, no virtual
CPU, no separate kernel booting underneath it. A "containerized" process
is an ordinary process on the host, running under the same kernel as
everything else, with a handful of kernel features layered on top to make
it *look* isolated: it can't see other processes, can't see the real
filesystem root, can't use up the whole machine's memory, and thinks its
hostname and network interfaces are its own. Docker, containerd, Podman,
and Kubernetes are all, underneath, orchestration on top of the same three
or four Linux kernel primitives — they don't implement isolation
themselves, they configure it.

## Namespaces: giving a process its own view of global state

Most kernel resources are global by default — there is one process table,
one filesystem root, one hostname, one set of network interfaces, visible
identically to every process on the box. A **namespace** doesn't create a
new resource; it creates a new, process-local *view* of one, via
`clone()` flags or the `unshare(2)`/`setns(2)` syscalls:

- **PID namespace** — processes inside see their own PID 1 and their own
  process tree; PID 400 on the host might be PID 1 inside the container,
  and processes outside the namespace are invisible from inside it.
- **Mount namespace** — a private view of the mount table, so mounting or
  unmounting a filesystem inside the container doesn't touch the host's
  view (and vice versa).
- **Network namespace** — its own network interfaces, routing table, and
  port space — this is what lets two containers both bind port 8080
  without conflicting.
- **UTS namespace** — its own hostname, so `hostname` inside a container
  doesn't return the host's.
- **IPC namespace** — its own System V IPC and POSIX message queue
  identifiers.
- **User namespace** — lets a process be "root" (UID 0) inside the
  namespace while mapping to an unprivileged UID outside it — the basis
  of rootless containers.

A container is, mechanically, a process started with most or all of
these namespaces unshared from the host at once. Nothing about namespaces
is container-specific — `unshare --pid --mount --net bash` gets you most
of the way to "container" by hand, with no image, no daemon, and no
Docker involved, which is the fastest way to see that the isolation is a
kernel feature Docker configures rather than one Docker invented.

## cgroups: limiting, not isolating

Namespaces answer "what can this process *see*?" A completely different
mechanism, **control groups** (cgroups), answers "how much can this
process *use*?" — CPU shares, memory ceilings, block I/O bandwidth, PID
count. A cgroup is a hierarchical grouping of processes with limits and
accounting attached, exposed as a virtual filesystem
(`/sys/fs/cgroup/...`) that both the kernel and userspace tools read and
write.

This is the difference that matters most when a container behaves
unexpectedly: a process that gets OOM-killed inside a container did not
run out of "container memory" — it hit a cgroup memory limit, enforced by
the same kernel mechanism whether or not Docker is involved, and the
container's dmesg-visible OOM event looks identical to a bare-metal
process hitting `ulimit`. Namespaces and cgroups are also independent of
each other: you can put a process in a memory cgroup without any
namespace isolation at all, which is exactly what `systemd` does for
ordinary system services, no containers in sight.

## The root filesystem: `chroot`, `pivot_root`, and image layers

A container also needs its own filesystem — not the host's `/`, but
whatever the image shipped: an Alpine or Debian userland, an application
binary, its libraries. The classic primitive is `chroot(2)`, which
changes what a process considers to be `/`; container runtimes actually
use `pivot_root(2)`, a stricter variant that fully swaps the root
filesystem and unmounts the old one, closing off escape routes that
`chroot` alone leaves open (a process with enough privilege can `chroot`
into a subdirectory and then walk back out via a retained file descriptor
to the old root).

The filesystem itself is typically **OverlayFS**: a union filesystem that
stacks read-only layers — one per image layer — under a single writable
layer on top. A `Dockerfile` instruction like `RUN apt-get install
curl` produces exactly one new layer, a diff of the filesystem before and
after that instruction. This is why Docker image layers are cached and
reordering `Dockerfile` instructions changes build speed: everything
above the first changed layer must rebuild, so put the parts that change
least often (base image, dependency installation) before the parts that
change most often (application source).

## The runtime stack: OCI, runc, and where Docker fits

"Docker" is actually several layered pieces, standardized industry-wide
via the **Open Container Initiative (OCI)** specs so that images and
runtimes from different vendors interoperate:

- An **OCI image** is a spec for the layers-plus-metadata format above —
  what `docker build` produces and what a registry like Docker Hub
  stores.
- An **OCI runtime** is a spec for actually running one of these,
  implemented by `runc` (the reference implementation, also used
  underneath containerd, CRI-O, and Podman): given an unpacked root
  filesystem and a config describing namespaces/cgroups/capabilities,
  `runc` makes the `clone()`/`unshare()`/cgroup-configuration calls
  described above and `exec`s the container's entrypoint.
- **containerd** sits above `runc`, managing image pulls, storage, and
  the lifecycle of many containers, and is what the Docker daemon
  (`dockerd`) itself delegates to — and what Kubernetes talks to directly
  via the Container Runtime Interface (CRI), with no Docker daemon
  involved at all on most modern clusters.

The practical upshot: "Docker" the CLI, "Docker" the daemon, and
"container" the running process are three different things, and
Kubernetes removing Docker-the-daemon as a runtime a few years back
(while still running perfectly normal OCI containers via containerd)
confused people precisely because those three things get conflated.

## Networking: veth pairs and bridges

A container's network namespace starts with nothing but a loopback
interface — no way to reach the outside world. The usual setup creates a
**veth pair**: two virtual Ethernet interfaces that act like a patch
cable, with one end moved into the container's network namespace and the
other left in the host's, attached to a bridge (`docker0` by default).
Traffic leaving the container's `eth0` arrives at the bridge as if from a
physical port, and the host does ordinary IP routing/NAT (`iptables`
rules Docker manages) from there to the outside network. Port publishing
(`-p 8080:80`) is a NAT rule, not a special container feature — it maps a
host port to the container's namespace-private port space through that
same bridge.

## Why containers aren't a security boundary the way VMs are

A VM's isolation is enforced by a hypervisor mediating access to actual
hardware, with each guest running its own kernel. A container's isolation
is enforced entirely by one shared kernel correctly restricting what each
namespace can see — which means a bug in namespace handling, a
misconfigured capability, or a container running as real root with a
mounted host path is a direct path to the host, because there's no second
kernel to escape through *in addition to* the first. This is why
`--privileged`, unconstrained bind mounts of `/`, and running containers
as UID 0 without a user namespace are treated as much bigger red flags
than the equivalent VM misconfiguration: the entire isolation model rests
on the kernel's bookkeeping being correct, not on a hardware-enforced ring
transition.

## Why this matters day to day

Most confusing container behavior traces back to one of the mechanisms
above once you know to look:

- **PID 1 problems** — the process a container starts as PID 1 inherits
  init's responsibilities (reaping zombie children, forwarding signals)
  whether or not it was written to do so; a plain `node` or `python`
  process as PID 1 often ignores `SIGTERM` on `docker stop` and leaves
  zombies, which is why `tini`/`dumb-init` or `--init` exist.
- **"Why can't my container see the host process/network/file"** — it's
  in a different namespace, by design; `--pid=host`, `--network=host`,
  and bind mounts are all ways of deliberately sharing one specific
  namespace back with the host instead of isolating it.
- **OOM kills that don't match host memory pressure** — a cgroup limit
  was hit, independent of how much memory the host actually has free.
- **Image bloat from a single late `Dockerfile` line** — every layer is
  additive; deleting a file in a later layer doesn't shrink the image,
  because the earlier layer with that file is still stored underneath it
  in the union filesystem.
