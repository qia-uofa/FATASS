# The `.class/` file

## Location

`<NodeDir>/.class/` — one per Node, alongside the rest of that Node's
(git-versioned) filesystem, where `<NodeDir>` is that Node's own
directory at whatever depth it lives (`./Topology/<Name>/` for a
top-level Node; `./Topology/<Group>/<Name>/`, `./Topology/<Parent>/<Name>/`
for one nested under a plain folder or inside another Node, and so on —
see `nesting.md`). Its *files* are named after `<NodeDir>`'s own final
segment — `<NodeName>` below — but the *class* they define is named
after `<NodeDir>`'s full path from `./Topology`, flattened with `_` (see
`nesting.md`'s "Naming"); the two only coincide for a top-level Node.

## Contents

`.class/` holds multiple Python modules:

- One defining the Node subclass itself:
  - The *file* is named `<NodeName>.py` — `<NodeDir>`'s own final
    segment. The *class* declared inside it is named after `<NodeDir>`'s
    full path, flattened with `_` (see `nesting.md`).
  - It subclasses `Node`, imported from the `agent` package. Every Node
    subclasses `Node` directly; there is no separate base class for a
    Node that hosts children (see `nesting.md`).
  - Its constructor takes no arguments — the working path is derived
    from the class's own name (`Path("Topology", *type(self).__name__
    .split("_"))` — see `nesting.md`) rather than passed in, which is
    what lets the same logic work at any nesting depth without
    inspecting the filesystem.
  - It defines `properties(self)` (see `properties.md`) and, if it has
    any transforms, `transform_names(self)` returning their names as
    plain strings — `Node.transforms` (inherited, not overridden) turns
    that list into instances lazily (see `transform.md`).
- One additional module per transform this Node uses, each defining a
  `class SpecificTransform(Transform)` (see `transform.md`).
- A shadow `agent.py` module: a hardcoded, verbatim copy of a file kept
  in the `agent` package itself, placed here purely so `.class/` files'
  `from agent... import ...` statements resolve even when the real
  `agent` package hasn't been pip-installed in the current environment.
  It does no work of its own beyond redirecting to the real package on
  disk. See "Shadow `agent` module" below.
- Description files: Markdown files storing the verbatim `--description`
  text a Node or Transform was created from, so it can later be checked
  against what `properties()`/`apply()` actually do (see
  `../Command Tree/audit.md`) instead of only surviving informally
  inside a generated docstring. `.class/<NodeName>.md` holds the Node's
  own description; `.class/<TransformName>.md` holds a given
  transform's. Both are written by `create` (see
  `../Command Tree/create.md`), kept in sync with their name by
  `rename`, and deleted by `remove` along with the file they describe
  (see `../Command Tree/rename.md`, `../Command Tree/remove.md`).

## Shadow `agent` module

`.class/*.py` files import from the real `agent` package. That package
is normally made importable via `pip install -e Agent`, but `.class/`
files are also read and, when justified, executed directly (see
`Agent/README.md`'s line-by-line execution rules), without that install
step necessarily having been run. To keep that path working regardless,
every `.class/` directory also carries a copy of a shadow `agent.py`
module — hardcoded once in the `agent` package (not generated per Node)
and copied in verbatim whenever `.class/` is created. Python always
searches a script's own directory first, so this copy resolves `import
agent` before it would otherwise fail; the shadow module then locates
the real package on disk by walking up its own ancestor directories one
at a time until it finds one containing `Agent/agent/`, and rebinds
itself to point at that, so submodule imports like `agent.core` resolve
to the real files. It walks rather than assuming a fixed number of
directories up, since a Node's `.class/` can sit at any nesting depth
under `./Topology` (see `nesting.md`). If the real package is already
importable through some earlier `sys.path` entry, that one wins and the
shadow is never reached.

## Remark

`.class/` itself — the Node and Transform class bodies, their structure
and control flow — is unaware of the Node's real content, context, or
theme. That awareness never gets hardcoded as deterministic logic; it
only enters through `Agent.free(...)` calls, which draw on the actual
filesystem and free-text descriptions at execution time.

## Lifecycle

- `create <NodePath>` (see `../Command Tree/create.md`) generates the
  Node subclass file when a Node is first created, implementing
  `properties(self)` from the description given at creation time,
  relying on the base class's empty default `transform_names()` (no
  override needed until the first transform exists), and copying in the
  shadow `agent.py` module described above.
- `create <TransformPath>` (see `../Command Tree/create.md`) adds a new
  transform module to `.class/` and adds its name to the owning Node
  subclass's `transform_names()` list (defining that method, overriding
  the base default, the first time it's needed).
- `rename` (see `../Command Tree/rename.md`) updates a Node's or
  Transform's name everywhere it's referenced across `.class/` — its own
  files, every other transform's `inputs`/`output` bindings, and
  `transform_names()` entries.
- `remove` (see `../Command Tree/remove.md`) deletes a Node's or
  Transform's `.class/` files once nothing else in the Topology still
  depends on them.
- `init` (see `../Command Tree/init.md`) diagnoses `.class/` for each
  existing Node — e.g. a missing or misnamed Node subclass file, a name
  in `transform_names()` with no matching module, a constructor that
  takes arguments — and resolves any issues found through dialogue with
  the user. It's also what creates `./Topology` itself the very first
  time it's missing — see `nesting.md`.
