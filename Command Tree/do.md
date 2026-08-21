# `do`

## Syntax

```text
>>> do "<python line>"
```

- `<python line>` — a single line of Python source, executed exactly as
  written. References any existing Node or Transform via the
  `agent.cwd`-relative resolvers `node(path)` / `transform(path)` (see
  `../Node/nesting.md`) — `node("Path/To/Node")` to instantiate a Node,
  `node("Path/To/Node").properties()` to inspect it, or
  `transform("Path/To/Node/.class/SpecificTransform.py").apply()` to run
  one of its registered transforms (see `../Node/transform.md`). Not
  limited to a single call chain on one Node — an assignment, a
  comparison, or a line touching more than one Node are all valid, as
  long as they fit on one line.
- Passed as a single quoted string.

### Example

```text
>>> do "transform('MyNode/.class/SpecificTransform.py').apply()"
```

Or, comparing two Nodes' status in one line:

```text
>>> do "node('MyNodeA').properties() == node('MyNodeB').properties()"
```

## Behavior

1. Parses `<python line>` and resolves every `node(path)`/`transform
   (path)` call it contains by joining `path` onto `agent.cwd` (see
   `../agent.md`) and reading the result under `./Topology` (see
   `../Node/nesting.md`) to find that Node's or Transform's class
   definition. Every path named in the line must resolve to something
   that already exists; an unresolvable path is an error surfaced as-is,
   not silently skipped.
2. Instantiates each resolved Node (no-argument constructor; working
   path already derived from its own class file's location — see
   `../Node/class.md`) and executes the line exactly as written against
   them — attribute/method access, assignment, comparison, whatever it
   contains.
3. Only Nodes and Transforms that actually exist under `./Topology` are
   in scope — not anything outside the Node object model entirely.
   `<python line>` may not `import` anything else or reach outside it
   either — it's a line evaluated against Nodes, not arbitrary code
   execution.
4. If the line invokes `transform(path).apply(...)` (or the equivalent
   `node(path).transforms['<TransformName>'].apply(...)`), that call's
   access is restricted per `../Node/transform.md`: edit access only to
   the transform's `output` Node, read-only to its `inputs`, and no edit
   access to any `.class/` directory.
5. Since this is invoked with `>>>`, it follows the verbatim-command
   contract: the line is run exactly as given — no inferring a different
   Node, method, or argument than what was typed — and any error it
   raises is surfaced as-is rather than caught and reinterpreted.
6. Whatever the line evaluates to (or produces as a side effect) is
   reported back to the user.

This is the direct, ad hoc interface to the Node object model — a way to
invoke a transformation, inspect `properties()`, or otherwise poke at
one or more Nodes on the spot, without a dedicated CLI subcommand
existing for it.
