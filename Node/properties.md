# `properties(self)`

## Signature

```text
def properties(self) -> str
```

## Purpose

Reports which of the Node's properties have been fulfilled and which
haven't, as a prompt string.

The property set itself isn't declared anywhere separate — it's implicit
in how this method is written. When `create <NodePath>` generates this
method, it derives that property set from the description given at
creation time (see `../Command Tree/create.md`) and bakes it into the
method body via `Agent.free(...)`, since deciding what counts as "the
properties" from free text is a judgment call rather than procedural
logic.

## Caching

When a Node's `properties()` includes an `Agent.free(...)` judgment call
(e.g. a qualitative "is this actually good enough" check that plain
existence/structure checks can't express), that call is not repeated on
every invocation — it's cached:

- Before invoking `Agent.free(...)`, compute a fingerprint of whatever
  inputs that judgment call actually reads (e.g. the mtimes/sizes of the
  files it's grounded in). Compare it against the fingerprint stored in
  `.class/.properties_cache` (created if missing).
- If the stored fingerprint matches the current one, return the cached
  judgment string instead of invoking `Agent.free(...)` again.
- If it doesn't match (or the cache file doesn't exist yet), invoke
  `Agent.free(...)`, then overwrite `.class/.properties_cache` with the
  new fingerprint and the judgment string it returned.
- The `agent` package provides this as a reusable helper —
  `agent.cache.cached_free(cache_path, fingerprint_inputs, task)` — so a
  Node's `properties()` calls that instead of `Agent().free(...)` directly
  wherever caching applies, rather than reimplementing the fingerprint
  logic per Node.

`.class/.properties_cache` is regenerable machine state, not source — it's
excluded from the outer repo via a dedicated `.gitignore` entry, even
though the rest of `.class/` is tracked. This only applies to the
`Agent.free(...)` portion of `properties()`; purely procedural checks
(file existence, structure) stay cheap already and don't need caching.

## Usage

- Called by `init` (see `../Command Tree/init.md`) while scanning existing
  Nodes, as part of diagnosing their current state.
- Callable directly through `do` (see `../Command Tree/do.md`), e.g.
  `>>> do "node('MyNode').properties()"`, to check a Node's status on
  demand.
- May be read by transformation methods (see `transform.md`) — e.g. a
  transform on one Node might inspect another Node's `properties()` output
  before deciding how to proceed.
