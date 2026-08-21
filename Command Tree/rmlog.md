# `rmlog`

## Syntax

```text
>>> rmlog
```

Takes no arguments.

## Behavior

1. Deletes `./log.md` if it exists. If it doesn't exist, this is not an
   error — same leniency `ExecutionLog`'s own construction already has
   (see `../README.md`'s "Execution log", `agent/log.py`): a missing log
   is just an empty one that hasn't been written to yet.
2. Never touches `./Topology` in any way. This is a log-file operation
   only, unrelated to the `>`/`>>`/`>>>` command surface's effect on
   Nodes (see `../README.md`'s "Access").
3. Like every command, `rmlog`'s own invocation still gets recorded (see
   "Execution log" below) — its heading, trace, and closing
   `**Overview:**` are written the same way any command's are. Since
   `ExecutionLog` recreates `./log.md` the moment anything is next
   appended to it, this write happens *after* step 1's deletion, so
   `rmlog`'s own section becomes the first thing in the freshly emptied
   file. `rmlog` clears history without breaking the invariant that every
   command execution gets recorded — it just starts that record over.

## Purpose

`./log.md` is append-only and never truncated by design (see
`../README.md`'s "Execution log") — the whole point is that no step of
any past command is ever silently lost, which is what makes `continue`
(see `continue.md`) trustworthy. But across a long-running project that
same discipline means the file only grows, without bound, forever.
`rmlog` is the one deliberate, explicit escape hatch from that
accumulation — a manual reset for when the file's size or its
accumulated clutter has become a burden of its own. Nothing else in the
command surface ever deletes or truncates `./log.md` on its own
initiative; this is the only command that does, and only because the
user asked for it directly.
