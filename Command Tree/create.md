# `create`

## Syntax

```text
>>> create <Path> --description "<description>" [--inputs <NodePath> [<NodePath> ...]]
```

- `<Path>` — what to create, relative to `agent.cwd` (see `../agent.md`).
  Its shape decides which of the two behaviors below runs; there's no
  separate flag for that part:
  - **Node path** — `<Path>` doesn't end in `.class/<Name>.py`. Creates
    a new Node at `Topology/<agent.cwd>/<Path>`. Every segment must be a
    valid Python class name (PascalCase, per the naming convention — see
    `../Node/nesting.md`), must not contain `_` (reserved as the
    flattened class name's separator), and, critically, may not be
    `.class` itself; any segment that doesn't exist yet is created as a
    plain organizational folder, except the last, which becomes the new
    Node's own directory. An error if the resolved path passes through an
    existing `.class/` directory anywhere along the way — `.class/` is
    reserved (see `../Node/nesting.md`) and can never contain a Node.
  - **Transform path** — `<Path>` ends in `.class/<TransformName>.py`
    (see `../Node/nesting.md`'s "Addressing" for the shape rule), and
    that's the *only* place a `.class` segment may legally appear in
    `<Path>`. Creates a new Transform registered on the Node at
    whatever's before `.class/` in `<Path>`. That owning Node must
    already exist; `<TransformName>` must be a valid Python class name.
- `--description` — free-text description of what the Node is for and
  what properties it should fulfill, or of what the Transform should do.
- `--inputs` — zero or more other Nodes to read from, each given as a
  `<NodePath>` relative to `agent.cwd`, same resolution as `<Path>`. For
  the Transform shape, these become the new Transform's `inputs`. For
  the Node shape, giving `--inputs` is shorthand for also creating a
  `"build"` transform on the new Node — see Behavior step 7 below.

This name-shape detection (Node path vs Transform path) is shared by
`remove` (`remove.md`), `rename` (`rename.md`), and `modify`
(`modify.md`) — all three reference this section rather than redefining
it.

## Behavior

### Creating a Node (`<Path>` doesn't end in `.class/<Name>.py`)

1. Rejects `<Path>` outright if any segment is `.class`, or if the path
   from `Topology/<agent.cwd>` down to (and through) any segment passes
   inside an existing `.class/` directory — `.class/` is reserved (see
   `../Node/nesting.md`) and this is the one thing `create` refuses
   before doing anything else. Otherwise, creates each segment of
   `<Path>` as a plain directory nested under `Topology/<agent.cwd>` (see
   `../agent.md`), creating any missing intermediate segment as an empty
   plain organizational folder (no `.class/`) along the way. The final
   segment's directory is the new Node's own directory. Not its own git
   repository — the whole Topology is versioned as part of the outer
   repo.
2. Creates the `.class/` subdirectory inside it, containing a Python
   file named after the Node's own final segment (e.g. `Style.py`),
   defining a class subclassing `Node` and named after the Node's *full*
   path from `./Topology`, flattened with `_` (e.g. `Draft_Control_Style`
   — see `../Node/nesting.md`), with a no-argument constructor (working
   path derived by splitting the class's own name back on `_`, not
   passed in — see `../Node/class.md`).
3. Implements `properties(self)` on that class so it reports which
   properties — as implied by `<description>` — are fulfilled and which
   aren't. The property set itself is derived from `<description>` and
   baked into the method (via `Agent.free(...)`, since deciding what
   counts as "the properties" from free text is a judgment call, not
   procedural logic).
4. That `Agent.free(...)` call's access is restricted: read-only access to
   every Node's `.class/` directory (useful for staying consistent with
   how existing Nodes are defined), and no read access to any Node's
   content outside `.class/`. Write access is confined to the new Node's
   own `.class/` — the new class file being created.
5. Copies the hardcoded shadow `agent.py` module (see `../Node/class.md`)
   into `.class/`, so files there can `import agent` without the real
   package needing to be pip-installed.
6. Writes `<description>` verbatim to `.class/<NodeName>.md` (`<NodeName>`
   here being the Node's own final segment, matching its class file's
   name — see `../Node/class.md`), so it can later be checked against
   what `properties()` actually does — see `audit.md`.
7. If `--inputs` was given, additionally creates a `"build"` transform
   on the new Node from the same `<description>` and `--inputs` —
   exactly the "Creating a Transform" behavior below, as if immediately
   followed by `create <NodePath>/.class/build.py --description
   "<description>" --inputs <NodePath...>`. This is the one reserved
   transform key `build` (see `build.md`) looks for, so a Node created
   with `--inputs` is immediately buildable without a separate `create`
   call.

Any Node created this way can host children of its own — and can itself
be created inside another Node's own directory, simply by `agent.cwd`
pointing there at creation time — without anything special being done to
enable it; every Node can, unconditionally (see `../Node/nesting.md`).

### Creating a Transform (`<Path>` ends in `.class/<TransformName>.py`)

1. Locates the owning Node's existing `.class/` directory — everything
   in `<Path>` before `.class/<TransformName>.py`, resolved against
   `agent.cwd`. If it's missing or broken, that's an `init` problem (see
   `init.md`) — resolve that first.
2. Adds a new module to `.class/` defining `class <TransformName>
   (Transform)` — this one name is used exactly as given, never flattened
   (only a Node's own class name is — see `../Node/nesting.md`) —
   following the transform contract (see `../Node/transform.md`): a
   no-argument constructor that hardcodes `inputs` (a `dict[str, Node]`
   built from every `--inputs` Node, keyed by its `./Topology`-relative
   path) and `output` (set to the owning Node), each resolved via
   `load_node_module(path)()` using a `./Topology`-relative path computed
   once now — from the `<NodePath>`s as typed relative to `agent.cwd` at
   this moment — rather than a plain import, plus an `apply(self)`
   method that overwrites `self.output`'s filesystem using `self.inputs`.
3. Implements `apply(self)` from `<description>`, using plain Python
   where a step is procedural, `Agent.free(...)` where it isn't, or a
   mix of both — per the same contract.
4. Every `Agent.free(...)` call made while generating that body has
   read-only access to every Node's `.class/` directory (useful for
   staying consistent with how existing Nodes/transforms are defined),
   and no read access to any Node's content outside `.class/`. Write
   access is confined to the owning Node's `.class/` — the new transform
   module being created. This is a separate restriction from the one
   governing `apply(self)`'s own eventual *execution* (see
   `../Node/transform.md`) — that one applies later, each time the
   generated transform actually runs.
5. Adds `"<TransformName>"` to the owning Node's `transform_names()`
   list, defining that method (overriding the base class's empty
   default) if it doesn't exist yet.
6. Writes `<description>` verbatim to `<OwningNodeDir>/.class/
   <TransformName>.md` (see `../Node/class.md`), so it can later be
   checked against what `apply()` actually does — see `audit.md`.
