# `init`

## Syntax

```text
>>> init
```

Takes no arguments. Operates on the current `agent.cwd` (see
`../agent.md`) — `.`, the root, the very first time it's ever run (before
any `cd`), or wherever `cd` last left it otherwise.

## Behavior

1. If `./Topology` doesn't exist yet, creates it. This is the one gap
   `init` closes that no other command tolerates missing — every other
   command assumes `./Topology` already exists by the time it runs.
2. Scans `Topology/<agent.cwd>` to detect this level's Nodes (see
   `../Node/nesting.md`'s "Enumerating a level"): descending
   transparently through any plain organizational folder, stopping at
   each Node found (its own nested children aren't descended into here —
   `cd` into it and run `init` again there, if wanted).
3. Diagnoses each detected Node's `.class/` definition — e.g. missing or
   malformed class file, class name not matching the Node's own full
   path flattened with `_` (see `../Node/nesting.md`), constructor
   issues, and the like.
4. If any issues are found, starts a dialogue with the user to resolve
   them before continuing.
5. Once this level is clean, enters a dialogue to suggest new Nodes to
   create or other follow-up actions to take.
