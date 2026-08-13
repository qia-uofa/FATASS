# Node

A Node (`<NodeDir>/`, a child of some Topology's `topology_path` — see
`topology.md`) is not fixed at package-build time. Nodes are
created during the runtime of the external agent — the `agent` package
only defines the `Node` base class and the tooling to create and manage
them; the actual set of Nodes under any given Topology is whatever the
external agent has built up over the course of its work.

Nodes typically come into existence via `create <NodeName>` (see
`../Command Tree/create.md`), or as a follow-up suggestion from the
`init` dialogue (see `../Command Tree/init.md`). They cease existing via
`remove <NodeName>` (see `../Command Tree/remove.md`), and are renamed
in place via `rename <NodeName> <NewName>` (see
`../Command Tree/rename.md`). A Node created with `create -t` is a
**topology node** — a Node that's also a `Topology`, hosting its own
nested Nodes (see `topology.md`).

See also:

- `class.md` — the `.class/` file
- `properties.md` — the `properties(self)` method
- `transform.md` — transformation methods
- `topology.md` — `Topology`, topology nodes, and the reserved root
