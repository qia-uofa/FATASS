# `free`

## Syntax

```text
>>> free <NodePath> "<description>"
```

- `<NodePath>` — the single Node this call is scoped to, relative to
  `agent.cwd` (see `../agent.md`), resolved the same way as every other
  command (see `../Node/nesting.md`).
- `<description>` — free text describing an arbitrary change to make
  within that Node.
- Always invoked with the strict `>>>` verbatim prefix, never `>` or
  `>>`. `free` carries none of the guardrails the other commands have
  beyond Node-scoping — it must never be something the agent infers its
  way into from prose or fuzzily-typed text; only a deliberate, exact
  `>>> free <NodePath> "..."` line invokes it.

## Behavior

`free <NodePath> "<description>"` *is* `agent.free(description)` (see
`../agent.md`), with `self` scoped to the resolved Node for the duration
of the call — there's no separate command-level indirection to describe;
the command's whole behavior is calling the method.

1. Hands `<description>` straight to `agent.free(...)`, with no
   Transform indirection — no going through `properties()` or
   `transforms[...].apply()`.
2. The agent reasons about `<description>` and edits whatever it needs
   to within the resolved Node to satisfy it — anything the fixed-shape
   commands below can't reach.
3. Scoped strictly to that Node: `free` has no access to any other Node
   anywhere under `./Topology` — it can neither read nor write anything
   outside it.
4. Within it, `.class/` is read-only, same as always — `free` may read
   it for context (e.g. to stay consistent with the Node's declared
   `properties()`/transforms) but never writes to it. Any Node nested
   inside its own directory (see `../Node/nesting.md`) is off-limits the
   same way: `free` may read it (e.g. to describe how many children it
   has) but never adds, removes, or edits anything inside it — that's
   structural, `create`'s/`remove`'s/`rename`'s job, not content `free`
   can touch.
5. Still bound by the general access rules (see `../README.md`,
   "Access"): `Blueprint/` and `Agent/` remain read-only regardless of
   what `<description>` asks for.
6. Whatever the natural-language task implies as a result is reported
   back to the user.

## Purpose

Every other command covers one specific, structured shape of Topology
edit: `create` adds new `.class/` definitions (a Node or a Transform,
depending on `<Path>`'s shape — see `create.md`); `remove` and `rename`
delete or rename them; `do`/`apply`/`build`/`update` invoke an existing
Transform's `apply()` against Nodes that already model the change being
made; `purge` deletes a Node's (optionally, recursively, its
build-dependencies') content without touching `.class/` at all (see
`purge.md`).

`free` is `agent.free` (see `../agent.md`) exposed directly as a
command — the escape hatch for whatever doesn't fit that shape within a
single Node — an edit to that Node's actual content that no existing
Transform expresses, without warranting a new one. It can't restructure
the Topology itself or touch another Node; for that, use the other
commands (`create <NodePath>` to add a Node, `create
<NodePath>/.class/<TransformName>.py` plus `apply`/`build`/`do` to move
content between Nodes through a proper Transform). Prefer the structured
commands whenever they fit; reach for `free` only for what they
genuinely can't express within the one Node it's scoped to.
