# Nesting and naming

## Filesystem

`./Topology` is a plain root directory — not itself a Node, not backed by
any `.class/` of its own. Everything else in this spec lives somewhere
underneath it.

A folder anywhere under `./Topology`, at any depth, is a **Node** if and
only if it contains a `.class/` subdirectory. A folder without `.class/`
is a plain organizational folder — it exists only to shape the path of
whatever Nodes live underneath it; it holds no `.class/` and no content
of its own.

A Node's own directory can contain, side by side: its `.class/`
definition, its own non-`.class/` content, and further nested subfolders
— each of which is either another Node (has its own `.class/`) or a
plain organizational folder leading to Nodes deeper down. Nesting is
purely a filesystem fact, not a distinct object-model type: any Node can
host children just by having folders placed inside it, and any plain
subfolder anywhere under `./Topology` can do the same. There is no
dedicated child-container subfolder and no flag that marks a Node as
capable of hosting children — every Node can, unconditionally.

```text
./Topology/Foo/.class/                 <- Foo, a top-level Node
./Topology/Group/Bar/.class/           <- Bar, nested under a plain folder Group
./Topology/Foo/Sub/.class/             <- Sub, nested directly inside Foo
```

A `.class/` directory itself is reserved: it holds only what `class.md`
describes (the Node's own class file, one module per transform, the
shadow `agent.py`, description `.md` files) and nothing else — never a
Node, never a plain organizational folder. Nodes nest inside a Node's
own directory, never inside its `.class/`; `Foo/.class/Bar/` is not a
valid location for anything, Node or otherwise. This is what keeps a
Transform path's shape unambiguous (see "Addressing" below): anything
found past a `.class/` segment is a transform module, full stop, never
another layer of nesting to descend into.

## Naming

Every folder name under `./Topology` follows Python class naming
convention (e.g. `MyNode`), and — unlike a plain organizational folder's
name, which is never used for anything but shaping a path — must not
contain `_`: a Node's **class name** is its full path from `./Topology`,
flattened by replacing each `/` with `_` (e.g. `./Topology/Draft/
Control/Style` names the class `Draft_Control_Style`), and allowing `_`
in a real segment would make that flattening ambiguous to read. This is
always the *class* name — the identifier declared inside the `.py` file
— never the file's own name (see "File names" below).

Because it's the Node's full path, a class name is unique across the
whole tree: no two Nodes can ever share one, unlike a bare folder name
(`Foo/Bar` and `Baz/Bar` are both valid folder layouts, but name
different classes, `Foo_Bar` and `Baz_Bar`).

### File names

A Node's `.class/` files are named after its own folder — the *last*
path segment only, not the flattened full name: the class file is
`.class/<own folder name>.py`, its description is
`.class/<own folder name>.md`. So `./Topology/Draft/Control/Style`'s
class file is `Draft/Control/Style/.class/Style.py`, and inside it:

```python
from agent import Node

class Draft_Control_Style(Node):
    ...
```

The file name and the class name it defines only coincide for a
top-level Node (no ancestors to fold in); at any depth below that they
diverge on purpose — the file name stays short and matches what you'd
expect to find inside that specific folder, while the class name alone
still tells you, unambiguously, exactly where in the tree it lives.

## Addressing

Nothing outside `do`'s raw Python (see `../Command Tree/do.md`) ever
refers to a Node by its class name — even though it's unique, spelling
out `Draft_Control_Style` on every command would defeat the point of
nesting things under short folder names in the first place. Every
command instead addresses a Node — and a Transform — by **relative
path**:

- A **Node path** is a relative filesystem path (POSIX-style, `.`/`..`
  segments allowed) from `agent.cwd` (see `../agent.md`) to the Node's
  own directory. Given `agent.cwd` is `Path`, the Node at
  `./Topology/Path/To/Node` is addressed as `To/Node`; from anywhere
  else it might be `../Path/To/Node`, `../../Path/To/Node`, and so on —
  ordinary relative-path resolution, the same as a shell's `cd`.
- A **Transform path** is a Node path with `.class/<TransformName>.py`
  appended — the relative path to the transform's own `.py` file. Given
  the same `agent.cwd`, the `Transform` transform on `Path/To/Node` is
  addressed as `To/Node/.class/Transform.py`.

A command distinguishes the two shapes structurally, not by counting
dots: a given path is a **Transform path** if its last two segments are
`.class/<Name>.py`; otherwise it's a **Node path**. A path with a
`.class` segment anywhere *other* than second-to-last is always invalid
— `.class/` is reserved (see "Filesystem" above), so nothing may be
addressed, or created, past one except that one trailing `<Name>.py`.
`create` (`../Command Tree/create.md`) defines this once; `remove`/
`rename` reference it rather than redefining it.

Whatever path a command is given, resolving it always follows the same
two steps:

1. Join it onto `agent.cwd` (relative segments included — `..` walks up,
   `.` is a no-op) to get a path relative to `./Topology`. An error if
   the result would escape above `./Topology` itself, or if any segment
   along the way doesn't exist.
2. For a Node path, that resolved path *is* the Node's directory. For a
   Transform path, the resolved path's parent-of-`.class` is the owning
   Node's directory, and the file itself is `<owning dir>/.class/
   <TransformName>.py`.

## How a Node finds its own path

Since a Node's class name is its own full path with `/` replaced by `_`
(see "Naming" above), its constructor can recover that path directly
from its own class name, with no filesystem introspection needed:

```python
class Node:
    def __init__(self):
        self.path = Path("Topology", *type(self).__name__.split("_"))
```

This keeps the constructor argument-free (see `class.md`) while working
uniformly at any nesting depth: `type(self).__name__` for a Node
subclass instance is always its class name, and since no real folder
name may contain `_` (see "Naming"), splitting it back into segments is
unambiguous in both directions.

## Resolving a path to a Node or Transform

Two families of helper, both defined in the `agent` package:

- `load_node_module(path)` / `load_transform_module(path)` — take a path
  already resolved relative to `./Topology` (step 1 above already done),
  and return the class defined at that location: a `Node` subclass for
  the former, `Transform` for the latter. `load_node_module` locates the
  file at `<path>/.class/<path's own last segment>.py` (see "File names"
  above), imports it, and returns `getattr(module, flattened_name)` --
  the class named after `path` flattened with `_`, per "Naming".
  `load_transform_module` locates the transform's own file directly
  (`path` already points at it) and returns the class inside unchanged
  by its name (a transform's own class name is never flattened -- only a
  Node's is, since a transform is only ever looked up within the one
  Node's own `transforms` dict it's registered under, never globally by
  bare name -- see `transform.md`). These are what generated `.class/*.py`
  code uses directly (see `transform.md`) -- a transform's `inputs`/
  `output` are baked in as `./Topology`-relative path strings at `create`
  time (see `../Command Tree/create.md`), so they stay correct no matter
  what `agent.cwd` is later, at the moment `apply()` actually runs.
- `node(path)` / `transform(path)` — the `agent.cwd`-aware convenience
  wrapping the above: resolve `path` against the current `agent.cwd`
  (the two steps above), then load and instantiate. These are what a
  human-typed reference actually goes through — every structured
  command's `<NodePath>`/`<TransformPath>` argument, and what `do`'s raw
  Python line calls directly (see `../Command Tree/do.md`) — since a
  path typed at the prompt is always relative to wherever `agent.cwd`
  currently is.

## Enumerating a level ("what's around here")

Some commands (`init`, `graph` — see `../Command Tree/init.md`,
`../Command Tree/graph.md`) need "the Nodes at this level" rather than
one Node by path. That's always relative to `agent.cwd`: starting from
`Topology/<agent.cwd>`, walk each direct subfolder except `.class`
itself (reserved — see "Filesystem" above, never descended into as if
it were an organizational folder); if a subfolder is a Node (has
`.class/`), it's one of this level's Nodes and descent stops there — its
own nested children aren't included, the same way a nested Node's
children were never part of an outer level's scan before. If it's a
plain organizational folder (no `.class/`), descend transparently
through it and keep looking below — it isn't a Node itself, so it never
appears in the result on its own, only whatever Nodes are eventually
found underneath it.

## `agent.cwd`

`agent.cwd: Path` (see `../agent.md`) is a plain relative path under
`./Topology`, defaulting to `.` (the root). Unlike the naming scheme it
replaces, `agent.cwd` isn't a minor convenience here — it's the base
every path argument in every command resolves against, `do`'s raw Python
included (via `node(path)`/`transform(path)` above). `cd` (see
`../Command Tree/cd.md`) is the only thing that changes it.

See also:

- `node.md` — how Nodes come to exist
- `class.md` — the `.class/` file
- `transform.md` — transformation methods, and how `inputs`/`output`
  resolve by path
- `../agent.md` — `agent.cwd`, and how every command reads it
- `../Command Tree/cd.md` — the command that changes it
