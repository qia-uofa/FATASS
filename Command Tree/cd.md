# `cd`

## Syntax

```text
>>> cd <path>
```

- `<path>` — one of:
  - `<NodeName>` — a topology node (see `../Node/topology.md`) that's a
    direct child of the current `agent.topology`. Moves `agent.topology`
    into that Node's own nested Topology.
  - `<NodeName1>/<NodeName2>/.../<NodeNameN>` — the same, chained: each
    segment must be a topology node that's a child of the Topology the
    previous segment moved into.
  - `..` — moves `agent.topology` to its parent Topology (the Topology
    hosting the topology node `agent.topology` currently is). An error
    if `agent.topology` is already the root — the root has no parent.
  - `,` — moves `agent.topology` straight back to the root Topology
    (`./Topology`, reserved class name `topology`), regardless of how
    deep the current `agent.topology` is nested.

## Behavior

1. For `<NodeName1>/.../<NodeNameN>`: resolves each segment in order,
   the first against the current `agent.topology.topology_path`, each
   subsequent one against the Topology the previous segment moved into —
   the same per-segment resolution `do`/`build`/etc. use (see `do.md`),
   applied once per segment.
2. Each resolved Node must be a topology node — its class inherits
   `Topology`, not just `Node` (see `../Node/topology.md`). Resolving a
   plain Node this way is an error, surfaced as-is: `cd` can't move into
   something with no nested Topology to move into.
3. For `..`: resolves the topology node whose own nested Topology *is*
   the current `agent.topology`, and sets `agent.topology` to the
   Topology that topology node is itself a child of. An error if
   `agent.topology` is already the root (see Syntax above).
4. For `,`: sets `agent.topology` directly to the root Topology instance
   (class `topology`, `./Topology`), independent of how deep the current
   `agent.topology` is nested.
5. On success, sets `agent.topology` (see `../agent.md`) to the resolved
   Topology instance. Every subsequent command's `<NodeName>` resolution
   is relative to it until the next `cd`.
6. Reports back the resolved Topology's path (e.g.
   `./Topology/Node1/Topology/.../NodeN/Topology`, or `./Topology` after
   `cd ,`).

### Example

```text
>>> cd MyNode
```

moves the agent to work under `./Topology/MyNode/Topology`. From there:

```text
>>> cd Sub1/Sub2
```

moves it to `./Topology/MyNode/Topology/Sub1/Topology/Sub2/Topology`.
Then:

```text
>>> cd ..
```

returns it to `./Topology/MyNode/Topology/Sub1/Topology`, and:

```text
>>> cd ,
```

returns it all the way to `./Topology`, the root.

## Purpose

Makes nested topology nodes (see `../Node/topology.md`) navigable the
same way a filesystem's directories are — every other command's
`<NodeName>` always means "a child of whatever Topology `cd` last left
`agent.topology` pointing at" (see `../agent.md`), so working deep
inside a nested Topology doesn't require spelling out the full nested
path on every single command.
