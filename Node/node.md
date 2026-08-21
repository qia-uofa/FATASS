# Node

A Node (`<NodeDir>/`, any folder under `./Topology` that contains a
`.class/` subdirectory — see `nesting.md`) is not fixed at package-build
time. Nodes are created during the runtime of the external agent — the
`agent` package only defines the `Node` base class and the tooling to
create and manage them; the actual set of Nodes under `./Topology` is
whatever the external agent has built up over the course of its work.

Nodes typically come into existence via `create <NodePath>` (see
`../Command Tree/create.md`), or as a follow-up suggestion from the
`init` dialogue (see `../Command Tree/init.md`). They cease existing via
`remove <NodePath>` (see `../Command Tree/remove.md`), and are renamed
in place via `rename <NodePath> <NewName>` (see
`../Command Tree/rename.md`). Every Node subclasses `Node` directly —
there is no separate type for a Node that hosts children; any Node can
have other Nodes (or plain organizational folders leading to further
Nodes) nested inside its own directory, simply by those folders existing
there (see `nesting.md`).

See also:

- `class.md` — the `.class/` file
- `properties.md` — the `properties(self)` method
- `transform.md` — transformation methods
- `nesting.md` — how nesting and class names work, and the reserved
  `./Topology` root
