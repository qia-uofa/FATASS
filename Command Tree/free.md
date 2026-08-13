# `free`

## Syntax

```text
>>> free <NodeName> "<description>"
```

- `<NodeName>` — the single Node this call is scoped to; resolved
  against the current `agent.topology` (see `../agent.md`), same as
  every other command.
- `<description>` — free text describing an arbitrary change to make
  within that Node.
- Always invoked with the strict `>>>` verbatim prefix, never `>` or
  `>>`. `free` carries none of the guardrails the other commands have
  beyond Node-scoping — it must never be something the agent infers its
  way into from prose or fuzzily-typed text; only a deliberate, exact
  `>>> free <NodeName> "..."` line invokes it.

## Behavior

`free <NodeName> "<description>"` *is* `agent.free(description)` (see
`../agent.md`), with `self` scoped to `<NodeName>` for the duration of
the call — there's no separate command-level indirection to describe;
the command's whole behavior is calling the method.

1. Hands `<description>` straight to `agent.free(...)`, with no
   Transform indirection — no going through `properties()` or
   `transforms[...].apply()`.
2. The agent reasons about `<description>` and edits whatever it needs
   to within `<NodeName>` to satisfy it — anything the fixed-shape
   commands below can't reach.
3. Scoped strictly to `<NodeName>`: `free` has no access to any other
   Node in the current `agent.topology` (or any other Topology) — it can
   neither read nor write anything outside `<NodeName>` itself.
4. Within `<NodeName>`, `.class/` is read-only, same as always — `free`
   may read it for context (e.g. to stay consistent with the Node's
   declared `properties()`/transforms) but never writes to it. If
   `<NodeName>` is a topology node (see `../Node/topology.md`), its
   nested `Topology/` is off-limits the same way: `free` may read it
   (e.g. to describe how many children it has) but never adds, removes,
   or edits anything inside it — that's structural, `create`'s/
   `remove`'s/`rename`'s job, not content `free` can touch.
5. Still bound by the general access rules (see `../README.md`,
   "Access"): `Blueprint/` and `Agent/` remain read-only regardless of
   what `<description>` asks for.
6. Whatever the natural-language task implies as a result is reported
   back to the user.

## Purpose

Every other command covers one specific, structured shape of Topology
edit: `create` adds new `.class/` definitions (a Node or a Transform,
depending on `<Name>`'s shape — see `create.md`); `remove` and `rename`
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
commands (`create <NodeName>` to add a Node, `create
<NodeName>.<TransformName>` plus `apply`/`build`/`do` to move content
between Nodes through a proper Transform). Prefer the structured commands
whenever they fit; reach for `free` only for what they genuinely can't
express within the one Node it's scoped to.
