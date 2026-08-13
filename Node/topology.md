# Topology

## Shape

`Topology` is a subclass of `Node`, defined in the `agent` package
alongside `Node` and `Transform`. Where a plain `Node` only ever holds
content and a `.class/` definition, a `Topology` additionally *hosts*
other Nodes — it's the class both the reserved root (see below) and any
**topology node** are instances of.

A topology node is a Node whose generated class inherits `Topology`
instead of `Node` directly (`class MyNode(Topology)`) — created via
`create -t <Name>` (see `../Command Tree/create.md`). Being a Node, it
has the usual `<Name>/.class/` definition and can be built, removed,
renamed, purged like any other Node. Being *also* a Topology, its own
directory additionally contains a `Topology/` subdirectory, a sibling of
`.class/`, which hosts that Node's own children:

```text
./Topology/MyNode/.class/         <- MyNode's own Node definition
./Topology/MyNode/Topology/       <- MyNode's nested Topology
./Topology/MyNode/Topology/Sub/   <- a child Node living inside MyNode
```

This nests arbitrarily: a Node under `MyNode/Topology/` can itself be a
topology node with its own further-nested `Topology/`, and so on.

The repo root follows the exact same shape, one level further out: the
outer repo's own root directory (`./`, sibling to `Blueprint/`,
`Agent/`, `build.py`, ...) plays the role `MyNode/` plays above, and
`./Topology` is *its* `Topology/` — the reserved root's nested children
folder:

```text
./.class/                         <- the root's own Node definition (class `topology`)
./Topology/                       <- the root's own nested Topology
./Topology/MyNode/.class/         <- MyNode's own Node definition
./Topology/MyNode/Topology/       <- MyNode's nested Topology
```

So `.class/` and `Topology/` are always siblings, at every level,
including the root's — see "The root Topology" below.

## `topology_path`

Every `Topology` instance exposes `topology_path: Path` — the directory
its children live directly under. There's exactly one rule, no
exceptions: `topology_path` is `self.path / "Topology"`. For an ordinary
topology node that's `<NodeDir>/Topology`; for the root it's `./Topology`,
since the root's `self.path` is the outer repo's root directory itself
(see below) — the same formula, not a special case.

`node_names()`, `node_path(name)`, `load_node_module(name)`,
`load_transform_module(node_name, transform_name)`, and
`diagnose_node(name)` (see `../Command Tree/init.md`) are all defined on
`Topology` and resolve relative to `self.topology_path`, never a
hardcoded global root — this is what makes the same logic work
identically for the repo-root Topology and for any nested one.

## How a Node finds its own path

Since a Node's directory can now sit at any nesting depth, `Node.__init__`
can no longer hardcode `path` as a fixed formula off a single global root
(`Path("Topology") / type(self).__name__`, as it did before topology
nodes existed). It also can't use a bare `Path(__file__)`: `__file__`
referenced inside a method is resolved to *the file that method is
written in* — since `Node.__init__` is written once, in `agent/node.py`,
`Path(__file__)` there would always mean `agent/node.py` itself, no
matter which subclass instance actually calls it.

What it needs instead is *the file the actual runtime class is defined
in* — Python's `inspect.getfile(type(self))` answers exactly that,
as opposed to "what file is this code written in." A Node subclass is
always defined in `<NodeDir>/.class/<NodeName>.py`, so
`Path(inspect.getfile(type(self))).resolve().parent.parent` — two
directories up from that file — is exactly `<NodeDir>`, at whatever
depth it actually lives, for any real, named subclass: any Node, any
topology node, or the reserved root's own `topology` class. This keeps
the constructor argument-free (see `class.md`) while working uniformly
at any nesting depth, the same way the shadow `agent.py` module already
locates the real package by walking up from its own file (see
`class.md`'s Shadow `agent` module section) rather than assuming a fixed
relative offset.

Applied to the root: for a `topology()` instance,
`inspect.getfile(type(self))` is `./.class/topology.py`, so
`.resolve().parent.parent` is the outer repo's own root directory
(`.`) — the same formula as any other Node, no special-casing at all.
`topology_path` (`self.path / "Topology"`) then comes out to
`./Topology` automatically, which is exactly the well-known root
directory every other part of this spec refers to.

Bare `Topology()` — instantiated directly rather than through a named
subclass, as `Topology().load_node_module(...)` is throughout this spec
(see `transform.md`) — is the one case `inspect.getfile(type(self))`
can't help with: `type(self)` there is `Topology` itself, defined inside
the `agent` package, not any real `.class/` file. For that case
specifically, `Topology.__init__` instead inspects *its caller's* frame
(`inspect.stack()[1]`) to find the file that actually invoked
`Topology()`, and derives `path`/`topology_path` from that file's
location the same way — so a transform's `.class/SpecificTransform.py`
calling `Topology()` bare still resolves to the Topology that transform
module itself lives under, whatever `agent.topology` happens to be
pointing at when `apply()` later runs (see `transform.md`).

## The root Topology

`./Topology` is hosted by a `Topology` instance too, not just a plain
directory sitting there on its own: the outer repo's own root directory
has a `.class/topology.py`, sibling to `./Topology` (see "Shape" above),
defining a class named `topology` — all-lowercase, the one Node name
exempt from the PascalCase convention, and reserved: both `create <Name>`
and `rename <Name> <NewName>` refuse `"topology"` (case-insensitively)
as a Node name, anywhere in the Topology — see `../Command Tree/create.md`,
`../Command Tree/rename.md`.

Unlike every other Node, the root has no enclosing Node directory of its
own to live inside — every other Node's identity is looked up as
`<some topology_path>/<NodeName>`, but the root isn't anyone's child, so
there's no name-keyed lookup that ever needs to find it by name in the
first place (see `../agent.md`: it's reached only via `agent.topology`
itself, never resolved as a `<NodeName>` argument). That's why its
`.class/` can sit directly in the outer repo's root directory instead of
inside a folder named after it — nothing depends on that folder being
named `topology`, only on it containing `.class/topology.py` and,
alongside it, `./Topology`.

Unlike every other Node's class file, `.class/topology.py` isn't derived
from a `--description` via `Agent.free(...)` — there's no `create`
invocation that produces the root, so there's nothing to derive it from.
It's a hardcoded template shipped inside the generated `Agent/` package
(alongside the shadow `agent.py` module — see `class.md`) that `init`
copies to `./.class/` the first time it runs against a repo that doesn't
have one yet (see `../Command Tree/init.md`).

## `Agent.topology`

Which `Topology` the agent is currently working under — initially the
root — is tracked on `Agent` as `self.topology: Topology` (see
`../agent.md`). Every command that takes a `<NodeName>` (or a `<Name>`
of either shape — see `../Command Tree/create.md`) resolves it against
`agent.topology.topology_path`, not a hardcoded `./Topology`; `cd` (see
`../Command Tree/cd.md`) is what moves `agent.topology` to a different
Topology instance.

See also:

- `node.md` — how Nodes (topology nodes included) come to exist
- `class.md` — the `.class/` file, and the shadow-module mechanism
  `.class/topology.py` reuses for the root
- `../agent.md` — `Agent.topology`, and how every command reads it
- `../Command Tree/cd.md` — the command that changes it
