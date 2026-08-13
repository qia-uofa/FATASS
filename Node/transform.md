# Transforms

## Shape

A transform is a class:

```python
from agent.topology import Topology

class SpecificTransform(Transform):
    def __init__(self):
        self.inputs: dict[str, Node] = {
            "OtherNode": Topology().load_node_module("OtherNode").OtherNode(),
            "AnotherNode": Topology().load_node_module("AnotherNode").AnotherNode(),
        }
        self.output = Topology().load_node_module("MyNode").MyNode()

    def apply(self):
        ...  # overwrites self.output's filesystem using self.inputs
```

- `Transform` is the base class, defined in the `agent` package alongside
  `Node`.
- Its constructor (no arguments, same convention as `Node`'s own) hardcodes
  two properties:
  - `inputs` — a `dict[str, Node]` of every Node whose filesystem this
    transform reads from, keyed by Node name. There can be more than one.
  - `output` — the single Node whose filesystem this transform overwrites.
    It's usually (but need not be) the same Node the transform is
    registered under (see Storage below).
- It implements `apply(self)`, which overwrites `self.output`'s filesystem
  using `self.inputs` — either programmatically (plain Python), by
  invoking `Agent.free(...)`, or a combination of both.
- `apply(self)`'s access is restricted accordingly: edit access only to
  `self.output`'s filesystem; `self.inputs` Nodes are read-only. Neither
  extends to `.class/` — `apply()` never writes to any Node's `.class/`
  directory, `self.output`'s own included, only to its non-`.class`
  content. Editing `.class/` is `create`'s/`remove`'s/`rename`'s job,
  not a transform's.
- `inputs`/`output` are resolved via `Topology().load_node_module(name)`
  rather than a plain `import` of the referenced Node class. Those Nodes
  usually live in other `.class/` directories entirely, and when `output`
  is the Node this transform is registered under, a plain import would be
  circular (that Node's own file would need to import this transform back
  — see Storage). Resolving by name sidesteps both problems; it costs
  nothing extra since `Node.__init__` doesn't do any real work besides
  setting `path` (see Storage below on why). `Topology()` here resolves
  to the Topology this transform's own Node is itself a child of — never
  a hardcoded global root, and independent of wherever `agent.topology`
  (see `../agent.md`) happens to point at the moment `apply()` actually
  runs, since `inputs`/`output` were fixed at `create trans` time (see
  `../Command Tree/create.md`), not re-resolved later (see `topology.md`).

## Storage

Each Node subclass exposes its transforms as a dict property, keyed by
transform name — this is what `create`, `remove`, `rename`, and `do` use
to look a transform up by name (see Usage below); it's independent of
`output`, which is what `apply()` actually writes to. The dict itself is
built lazily, from a list of names a subclass declares:

```python
class MyNode(Node):
    def transform_names(self) -> list[str]:
        return ["SpecificTransform"]
```

`Node.transforms` (defined once on the base class, not overridden per
Node) turns that list into instances on first access, loading each
`<name>.py` module from this Node's own `.class/` directory via
`Topology().load_transform_module(...)` (same `Topology()` resolution as
`inputs`/`output` above — see `topology.md`). It is deliberately **not** built
eagerly in `Node.__init__`: since a transform's `output` is usually the
very Node it's registered under, eager construction would recurse forever
— instantiating a Node would build its transforms, each of which
instantiates that Node again as `output`, which would build its
transforms again, without end. Nothing ever needs another Node's
`.transforms` (only its `.path` — see `apply()` usage above), so leaving
it unbuilt until something actually asks for it breaks that cycle for
free.

## Definition location

Each `SpecificTransform(Transform)` subclass is defined in its own file
inside the owning Node's `.class/` directory, alongside the Node subclass
file itself (see `class.md`).

## Usage

- Invoked through `apply <NodeName> <TransformName> [args...]` (see
  `../Command Tree/apply.md`), or directly through `do` (see
  `../Command Tree/do.md`), e.g.
  `>>> do "MyNode().transforms['SpecificTransform'].apply()"`.
- May consult a target Node's `properties(self)` (see `properties.md`)
  beforehand, to decide what work is still needed.
- Created via `create <NodeName>.<TransformName>` (see
  `../Command Tree/create.md`); removed via `remove
  <NodeName>.<TransformName>` (see `../Command Tree/remove.md`);
  renamed via `rename <NodeName>.<TransformName> <NodeName>.<New>` (see
  `../Command Tree/rename.md`).
