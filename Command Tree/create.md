# `create`

## Syntax

```text
>>> create [-t] <Name> --description "<description>" [--inputs <Node> [<Node> ...]]
```

- `<Name>` — what to create. Its shape decides which of the two
  behaviors below runs; there's no separate flag for that part:
  - `<NodeName>` (no dot) — creates a new Node. Must be a valid Python
    class name (PascalCase, per the Topology naming convention), and
    may not be `topology` (case-insensitively) — that name is reserved
    for the root Topology (see `../Node/topology.md`).
  - `<NodeName>.<TransformName>` (one dot) — creates a new Transform
    registered on an existing Node. `<NodeName>` must already exist;
    `<TransformName>` must be a valid Python class name. Anything past
    the first dot is an error (a Transform can't be nested).
- `-t` — only valid alongside the Node shape of `<Name>` (an error
  combined with the dotted Transform shape): creates a **topology
  node** instead of a plain Node — see Behavior step 7 and
  `../Node/topology.md`.
- `--description` — free-text description of what the Node is for and
  what properties it should fulfill, or of what the Transform should do.
- `--inputs` — zero or more other Node names to read from. For the
  Transform shape, these become the new Transform's `inputs`. For the
  Node shape, giving `--inputs` is shorthand for also creating a
  `"build"` transform on the new Node — see Behavior step 8 below.

Every `<NodeName>` here — the one being created, any given via
`--inputs`, or the one a Transform is registered on — is resolved as a
child of the current `agent.topology` (see `../agent.md`), never a
hardcoded `./Topology`.

This name-shape detection (`<NodeName>` vs `<NodeName>.<TransformName>`)
is shared by `remove` (`remove.md`) and `rename` (`rename.md`) — both
reference this section rather than redefining it.

## Behavior

### Creating a Node (`<Name>` has no dot)

1. Creates `<Name>` as a plain directory under the current
   `agent.topology.topology_path` (see `../agent.md`) — `./Topology/<Name>`
   before any `cd`, or wherever `agent.topology` currently points — per
   the Topology filesystem convention (not its own git repository — the
   whole Topology is versioned as part of the outer repo).
2. Creates the `.class/` subdirectory inside it, containing a Python
   class named `<Name>` that subclasses `Node` — or `Topology` if `-t`
   was given, see step 7 — with a no-argument constructor (working path
   derived from the class file's own location on disk, not passed in —
   see `../Node/class.md`).
3. Implements `properties(self)` on that class so it reports which
   properties — as implied by `<description>` — are fulfilled and which
   aren't. The property set itself is derived from `<description>` and
   baked into the method (via `Agent.free(...)`, since deciding what
   counts as "the properties" from free text is a judgment call, not
   procedural logic).
4. That `Agent.free(...)` call's access is restricted: read-only access to
   every Node's `.class/` directory (useful for staying consistent with
   how existing Nodes are defined), and no read access to any Node's
   content outside `.class/`. Write access is confined to `<Name>/.class/`
   — the new class file being created.
5. Copies the hardcoded shadow `agent.py` module (see `../Node/class.md`)
   into `.class/`, so files there can `import agent` without the real
   package needing to be pip-installed.
6. Writes `<description>` verbatim to `.class/<Name>.md` (see
   `../Node/class.md`), so it can later be checked against what
   `properties()` actually does — see `audit.md`.
7. If `-t` was given, the class from step 2 subclasses `Topology`
   instead of plain `Node` (see `../Node/topology.md`), and this step
   additionally creates a `Topology/` subdirectory inside `<Name>`,
   sibling to `.class/` — the new topology node's own nested Topology,
   initially empty. Without `-t`, this step does nothing and `<Name>` is
   a plain Node with no nested Topology to `cd` into.
8. If `--inputs` was given, additionally creates a `"build"` transform
   on `<Name>` from the same `<description>` and `--inputs` — exactly
   the "Creating a Transform" behavior below, as if immediately followed
   by `create <Name>.build --description "<description>" --inputs
   <Node...>`. This is the one reserved transform key `build` (see
   `build.md`) looks for, so a Node created with `--inputs` is
   immediately buildable without a separate `create` call. Applies the
   same whether or not `-t` was also given — a topology node can have a
   `"build"` transform too, same as any other Node.

### Creating a Transform (`<Name>` is `<NodeName>.<TransformName>`)

1. Locates `<NodeName>`'s existing `.class/` directory. If it's missing
   or broken, that's an `init` problem (see `init.md`) — resolve that
   first.
2. Adds a new module to `.class/` defining
   `class <TransformName>(Transform)`, following the transform contract
   (see `../Node/transform.md`): a no-argument constructor that
   hardcodes `inputs` (a `dict[str, Node]` built from every `--inputs`
   Node, keyed by name) and `output` (set to `<NodeName>`), each resolved
   via `Topology().load_node_module(...)` rather than a plain import,
   plus an `apply(self)` method that overwrites `self.output`'s
   filesystem using `self.inputs`.
3. Implements `apply(self)` from `<description>`, using plain Python
   where a step is procedural, `Agent.free(...)` where it isn't, or a
   mix of both — per the same contract.
4. Every `Agent.free(...)` call made while generating that body has
   read-only access to every Node's `.class/` directory (useful for
   staying consistent with how existing Nodes/transforms are defined),
   and no read access to any Node's content outside `.class/`. Write
   access is confined to `<NodeName>/.class/` — the new transform module
   being created. This is a separate restriction from the one governing
   `apply(self)`'s own eventual *execution* (see `../Node/transform.md`)
   — that one applies later, each time the generated transform actually
   runs.
5. Adds `"<TransformName>"` to `<NodeName>`'s `transform_names()` list,
   defining that method (overriding the base class's empty default) if
   it doesn't exist yet.
6. Writes `<description>` verbatim to
   `<NodeName>/.class/<TransformName>.md` (see `../Node/class.md`), so
   it can later be checked against what `apply()` actually does — see
   `audit.md`.
