---
icon: lucide/bug
---

# Debuggers

A debugger looks like magic from the outside: point it at a running
process, and suddenly you can pause it mid-instruction, inspect its
registers, walk its stack, and step one line at a time. None of that is
magic. It's one process (the *tracer*) using an OS-provided control
channel to suspend, inspect, and mutate another process (the *tracee*),
plus a pile of debug metadata that maps raw memory addresses back to the
source lines a human wrote. Take away the metadata and a debugger still
works — it just shows you registers and hex instead of variable names and
`file.c:42`.

## The core primitive: ptrace

On Linux, essentially every debugger — gdb, lldb, strace, rr — is built on
one syscall: `ptrace(2)`. A tracer calls `ptrace(PTRACE_ATTACH, pid, ...)`
(or the tracee calls `ptrace(PTRACE_TRACEME, ...)` before `exec`-ing
itself, which is what happens when you run `gdb ./app` instead of
attaching to something already running). From that point on, the kernel
delivers every signal the tracee receives to the tracer first, and the
tracer decides what happens next: let the signal through, suppress it, or
use the pause as an opportunity to inspect and modify the tracee's state
via more `ptrace` calls — `PTRACE_PEEKTEXT`/`PTRACE_POKETEXT` to read and
write memory, `PTRACE_GETREGS`/`PTRACE_SETREGS` to read and write the
register file.

This is why only one tracer can attach to a process at a time — the
relationship is exclusive, tracked by the kernel — and why a process
being `ptrace`d by gdb can't simultaneously be `ptrace`d by strace. It's
also why containers and hardened kernels frequently break "just attach
gdb": `ptrace` is a raw capability to read and rewrite another process's
memory, so it's gated behind `CAP_SYS_PTRACE` and, on most modern distros,
a further restriction in `/proc/sys/kernel/yama/ptrace_scope` that limits
attaching to processes outside your own direct process tree unless you're
root. "Operation not permitted" from gdb is almost always one of these two
gates, not a bug in gdb.

## Software breakpoints: rewriting the instruction stream

Setting a breakpoint at a source line doesn't involve any special CPU
support at all, on most architectures. The debugger:

1. Resolves the source line to a memory address (see below).
2. Reads and saves the original byte at that address.
3. Overwrites it with a trap instruction — on x86, `0xCC`, the one-byte
   encoding of `INT3`.
4. Resumes the tracee.

When execution reaches that address, the CPU executes `INT3`, which
raises `SIGTRAP`. The kernel stops the tracee and — because it's being
traced — delivers the signal to the tracer instead of the tracee's own
signal handlers. The debugger now has control: it rewinds the tracee's
instruction pointer back by one byte (past the `INT3` it injected),
restores the original byte so the instruction stream is intact again, and
presents the pause to you as "stopped at breakpoint." When you continue,
it single-steps over the real instruction, re-injects the `0xCC`, and
resumes normally — so the next hit works too.

This is also why breakpoints don't survive across separately compiled
binaries, why setting a breakpoint on code that's about to be
`mprotect`'d read-only can fail, and why self-modifying or JIT-generated
code is uniquely painful to debug: the debugger is quietly editing the
same bytes the program might also be editing.

## Hardware breakpoints and watchpoints

`INT3` requires modifying code, which doesn't work for read-only pages,
and can't express "stop when this *variable* changes" without checking
after every single instruction. For that, CPUs expose dedicated debug
registers — `DR0`–`DR3` plus a control register on x86 — that the debugger
programs with an address and a condition (execute, write, or
read/write) directly in hardware. The CPU itself raises a trap when the
condition is met, with no instruction rewriting involved. This is what a
"watchpoint" in gdb (`watch some_variable`) usually compiles down to, and
it's why hardware watchpoints are limited in number (typically four on
x86) — you're bound by how many debug registers the silicon actually has.

## Stepping

"Step one line" and "step one instruction" both reduce to the same
mechanism as breakpoints, applied transiently. `PTRACE_SINGLESTEP` (or
setting the CPU's trap flag directly) tells the kernel to let exactly one
instruction execute and then deliver `SIGTRAP` again automatically — no
`INT3` injection needed for this case, since it's a per-instruction mode
rather than an address-triggered one. "Step over a line of C" is just this
primitive run in a loop, checking after each instruction whether the
debugger's line-number table (below) says you've reached a new source
line yet, and additionally not stepping *into* call instructions ("step
over") by placing a temporary breakpoint just after the call and running
to it instead of single-stepping through the callee.

## Symbols and debug info: getting source lines back

None of the above explains how a debugger knows that address `0x401136`
is `main.c:42`, or that the eight bytes at `rbp-0x18` are a variable named
`count`. That mapping lives in debug info generated by the compiler, in a
format like DWARF (used by GCC/Clang on Linux and macOS) or PDB (MSVC on
Windows). DWARF, specifically, ships a **line number program** — not a
table, but a tiny bytecode interpreter's worth of instructions that,
executed against a state machine, produce a table of (address, file,
line, column) rows. Compressing it as a program instead of a flat table
is what keeps debug info from being larger than the code it describes.

Alongside line numbers, DWARF encodes type and variable-layout
information: the shape of `struct Point`, which register or stack offset
holds a given local at a given program counter, and how to unwind one
stack frame to find the caller's — the basis for "stack trace" in every
debugger and most crash reporters. This is why stripping a binary
(`strip` on ELF, deleting the `.pdb` on Windows) doesn't change how it
runs — the debug info is metadata alongside the code, not part of the
executable logic — but turns a debugger's output from `main.c:42` back
into a bare address and, for stack traces, a pile of `??`.

## Why "attach" doesn't always work

A few failure modes recur across every platform, all variations on the
same theme — a debugger asking for a level of control the OS doesn't hand
out for free:

- **Linux, `ptrace_scope`**: a non-root, non-parent process usually can't
  attach without `CAP_SYS_PTRACE` or the target explicitly allowing it
  (`prctl(PR_SET_PTRACER, ...)`). Common in Docker (`--cap-add=SYS_PTRACE`
  is the usual fix) and on desktop distros that ship `ptrace_scope=1` by
  default.
- **macOS, System Integrity Protection**: SIP blocks debugging
  Apple-signed system binaries outright, and even user binaries need
  `com.apple.security.get-task-allow` (a Hardened Runtime entitlement) or
  running with `sudo` plus the debugger itself being appropriately signed,
  because macOS mediates debugging through Mach **task ports**
  (`task_for_pid`) rather than `ptrace`.
- **Windows**: the debugging API is a first-class part of Win32
  (`DebugActiveProcess`, `WaitForDebugEvent`), gated by the `SeDebugPrivilege`
  privilege — which admin accounts hold by default, which is part of why
  Windows debugging feels comparatively frictionless compared to a
  freshly hardened Linux box.

Different mechanisms, same shape: attaching a debugger is granting one
process the ability to read and rewrite another's memory and execution
state, and every OS treats that as a privileged operation with its own
gate to pass.

## Why this matters if you only ever click "Run and Debug"

IDE debuggers hide every piece of this behind a play button, but the
failures leak through anyway: a breakpoint that shows as a hollow circle
instead of solid red means the debugger couldn't resolve it to an address
— usually a symbol/debug-info mismatch between the binary you're running
and the source you're looking at. "Attach to process" greyed out or
failing usually means one of the OS-level gates above, not a problem with
your code. And a stack trace full of `<optimized out>` or missing frames
is DWARF/PDB information the optimizer legitimately destroyed — variables
that never got a stack slot, or a frame the compiler inlined away —
rather than a bug in the debugger itself. Knowing the mechanism turns
these from mysterious IDE quirks into specific, diagnosable questions.
