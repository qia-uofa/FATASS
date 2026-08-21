# Topology

## Filesystem

The agent works in a filesystem rooted at `./Topology`, where `./` is the
directory the external agent itself is running in. `./Topology` itself
is a plain directory — not a Node, not backed by any `.class/` of its
own.

Any folder anywhere under `./Topology`, at any depth, is a **Node** if it
contains a `.class/` subdirectory — that's the whole test, checked by
presence, not by any separate registry. A folder without `.class/` is a
plain organizational folder: it holds no content of its own, existing
only to shape the path of whatever Nodes sit underneath it. A Node's own
directory can likewise contain further nested subfolders — Nodes or
plain organizational folders — alongside its `.class/` and its own
content; nesting is purely a filesystem fact, not a distinct
object-model type. Every folder name under `./Topology` follows Python
class naming convention (e.g. `MyNode`).

Each Node folder is a plain directory — not its own git repository. The
whole Topology is versioned as part of the outer repo like everything
else (`.class/` tracked, non-`.class/` content ignored per the outer
`.gitignore`; see `../README.md`), rather than each Node keeping
independent history. (An earlier version of this spec had each Node
initialize its own nested git repo; that made `.class/` impossible for
the outer repo to track at all — a nested `.git` makes git treat the
whole folder as an untrackable embedded repository — without also
registering it as a proper submodule, which nothing here ever did. Dropped
as unnecessary complexity that didn't actually work.)

### Naming and addressing

A Node's *class* name is its full path from `./Topology`, flattened by
replacing each `/` with `_` — e.g. `./Topology/Draft/Control/Style`
names the class `Draft_Control_Style`, unique across the whole tree. Its
`.class/` *files*, though, are named after its own folder only (the last
segment) — `Draft/Control/Style/.class/Style.py` contains `class
Draft_Control_Style(Node)` — so a short file holds a long, unambiguous
class name.

No command spells that class name out, though: every command addresses
a Node — and a Transform — by **relative path** from `agent.cwd` (see
`agent.md`) instead. A Node path is the relative path to its own
directory (e.g. `To/Node`, `../Other/Node`); a Transform path is a Node
path with `.class/<TransformName>.py` appended (e.g.
`To/Node/.class/Transform.py`) — the relative path to the transform's
own `.py` file. See `Node/nesting.md` for the full mechanics.

## Access

`./Topology` is only ever edited as a side effect of running a command —
one of the `>`/`>>`/`>>>`-prefixed lines below, `free` included; plain,
unprefixed conversation never edits anything (see "Command input
contract" below for the full prefix contract). `Blueprint/` and `Agent/`
are read-only from every command's perspective — read for context, never
written to. Regenerating `Agent/` from `Blueprint/` (see `../build.md`)
is a separate, one-off meta-process, not part of the `>`/`>>`/`>>>`
command surface.

## Command input contract

How the agent reading `Agent/README.md` (the generated operating manual)
must interpret a line of user input at the prompt:

- A line with no `>` at all is plain conversation — the agent must
  never infer or run a command from it, no matter how command-like it
  reads. If it looks like the user is asking for a command, the agent
  says so and asks them to prefix the line with `>`, rather than
  guessing on their behalf.
- A line starting with a single `>` is a **command prompt** — a
  natural-language instruction that prompts a command, not command
  syntax itself. Infer the command/args it implies and prompt the user
  to confirm before proceeding.
- A line starting with `>>` is a **fuzzy command** — text already
  shaped like a command invocation, just imprecise (a typo, off
  casing, an approximate flag). Infer the closest matching real
  command/args and prompt the user to confirm before proceeding — same
  infer-then-confirm mechanics as `>`, just starting from command-shaped
  input instead of free prose.
- A line starting with `>>>` is a **verbatim command** — execute it
  exactly as written, even if it produces an error. This is strict: the
  agent must not assume a typo, "helpfully" reinterpret the command, or
  silently substitute what it thinks was meant. Run what's there and
  surface whatever error results, unchanged.
- Only input carrying one of these three prefixes may result in an edit
  to `./Topology`. Plain conversation, having none, can never edit
  anything — that's the whole point of routing it back to `>` instead of
  acting on it.

## Execution model

"Execution" means the agent works through the package's pseudocode via
its own reasoning (LLM thinking), not by literally running Python. If
the agent does have real Python execution available and it's justified
(e.g. a deterministic computation or parse step), it may run that
portion of the pseudocode for real instead of simulating it.

While executing, the agent must work through the underlying pseudocode
one line at a time, using only that line's local state — never drawing
on earlier conversational context for a procedural step. The only place
prior context legitimately re-enters is inside an `Agent.free(...)` call,
where natural-language judgment may draw on anything relevant;
everywhere else, follow only the current line.

## Execution log (`./log.md`)

`./log.md` (a Markdown file at the working directory root, sibling to
`./Topology`; created if missing, append-only, never truncated) records
execution as a sequence of sections, one per command, each opened by a
level-2 heading:

- Command heading: every command invocation opens a new `##` section
  titled with the command exactly as run, as inline code with the
  `>>>` prefix (e.g. ``## `>>> build Overview` ``), regardless of
  whether the triggering input was `>`, `>>`, or `>>>`. By execution
  time, `>`/`>>` input has already been resolved and confirmed into an
  exact command, so the heading always records that resolved form in
  its canonical `>>>` shape — never a `command:` label, never an
  annotation on how it was inferred.
- Trace item: every traced pseudocode line appends one `-` list item
  under the current heading, with two parts on two lines — a link to
  that line (e.g. `[topology.py:52](Agent/agent/topology.py#L52)`),
  then on the next line that line's verbatim source text in a fenced
  code block.
- Granularity: every line inside the command's own call chain
  (`Agent/agent/**`, and any target Node's `.class/**`) gets its own
  trace item — this is not discretionary. When a loop invokes
  *structurally identical* code once per item (e.g. the same
  `diagnose_node()` body run once per Node name), the trace-item
  *count* must still match the iteration count exactly — one item (or
  set of items) per iteration, never one item standing in for several.
  Only the item's annotation text may be shared/abbreviated across
  iterations; the per-iteration item itself is never optional.
- Free-input item: for a traced line that calls `Agent.free(...)`,
  before that call is carried out — after its trace item, before its
  result exists — one `- **free-input:**` list item logging the exact
  `task` string passed to it, as inline code. Logged before doing the
  thing, not after.
- Overview paragraph: after a command's last list item, one
  `**Overview:** ...` paragraph closing the section, summarizing the
  change or output the command produced.

Each entry must be appended *at the moment* its step executes —
interleaved with execution, one entry per step as the agent goes — never
written retrospectively (e.g. reconstructed from memory and batch-
appended once a command "finishes"). Writing the entry is part of
performing the step, not a record of it after the fact: that's what
forces every step to actually happen, in order, with nothing silently
skipped or summarized away. The overview paragraph is the one entry that
can only be written once a command's steps are done, but it still
reports what actually happened, not a plan, and is written the instant
that final summarizing step executes.

## Build target

This blueprint's generated package output (`$OUTPUT` in `../build.md`)
is `../Agent`.

## The Node class

Inside `.class/<own folder name>.py`, a Python class is defined:

- Its name is the Node's full path from `./Topology`, flattened with `_`
  — not the file's own name, which stays just the folder's own final
  segment (see "Naming and addressing" above, `Node/nesting.md`).
- It subclasses `Node`, defined in the `agent` package. Every Node
  subclasses `Node` directly, whether or not it hosts children of its
  own.
- Its constructor takes no arguments — the working path is derived from
  the class's own name (splitting it back into segments on `_`), which
  is what lets the same logic work at any nesting depth without
  inspecting the filesystem (see `Node/class.md`, `Node/nesting.md`).

### `properties(self)`

Returns a prompt string reporting which of the Node's properties have been
fulfilled and which haven't. The objective properties themselves are
implicit in how this method is written — they aren't declared elsewhere.
This method may call `Agent.free(...)` to generate the report.

### Transforms

A Node class also carries a `transforms` dictionary property, mapping
transform names to instances of `Transform` subclasses (one per name,
each defined in its own module in `.class/`, alongside the Node subclass
itself). This dict is built lazily, on first access, from a
`transform_names(self)` method a Node subclass declares (just the names —
resolving them into instances happens only when something actually reads
`transforms`, not eagerly in `__init__`): a transform's `output` is
usually the very Node it's registered under, so building it eagerly would
recurse forever (see `Node/transform.md`).

Each transform's no-argument constructor hardcodes two properties:
`inputs`, a `dict[str, Node]` of every Node it reads from, keyed by that
Node's `./Topology`-relative path (there can be more than one input),
and `output`, the single Node whose filesystem it overwrites — both
resolved via `load_node_module(...)` rather than a plain import, since
the referenced Nodes usually live in other `.class/` directories
entirely. It implements `apply(self)`, which overwrites `output`'s
filesystem using `inputs` — either programmatically (plain Python), by
invoking `Agent.free(...)`, or a combination of both.

Remark: `.class/` itself — the Node and Transform class bodies, their
structure and control flow — is unaware of the Node's real content,
context, or theme. That awareness never gets hardcoded as deterministic
logic; it only enters through `Agent.free(...)` calls, which draw on the
actual filesystem and free-text descriptions at execution time.

See also:

- `Node/node.md` — how Nodes come to exist
- `Node/class.md` — the `.class/` file itself
- `Node/properties.md` — the `properties(self)` method
- `Node/transform.md` — transformation methods
- `Node/nesting.md` — nesting, class names, and the `./Topology` root
- `agent.md` — `Agent`'s own state (`agent.cwd`) and how commands read it

## Commands

- `init` — see `Command Tree/init.md`
- `cd` — see `Command Tree/cd.md`
- `create` — see `Command Tree/create.md`
- `remove` — see `Command Tree/remove.md`
- `rename` — see `Command Tree/rename.md`
- `move` — see `Command Tree/move.md`
- `modify` — see `Command Tree/modify.md`
- `do` — see `Command Tree/do.md`
- `apply` — see `Command Tree/apply.md`
- `build` — see `Command Tree/build.md`
- `update` — see `Command Tree/update.md`
- `purge` — see `Command Tree/purge.md`
- `graph` — see `Command Tree/graph.md`
- `free` — see `Command Tree/free.md`
- `audit` — see `Command Tree/audit.md`
- `continue` — see `Command Tree/continue.md`
- `rmlog` — see `Command Tree/rmlog.md`
