# `continue`

## Syntax

```text
>>> continue
```

Takes no arguments -- always operates on the single most recent
unfinished command recorded in `./log.md` (see `../README.md`'s
"Execution log" section for that file's format).

## Behavior

1. Reads `./log.md` from the end and finds the last `##`-headed section
   that has no `**Overview:**` paragraph before either the next `##`
   heading or end of file -- an *unfinished* command, left mid-execution
   (e.g. the executing agent's session ended before it reached its own
   closing step). If every section already has its `**Overview:**`, this
   is an error: nothing to continue, and nothing is appended to the log.
2. Reads that section's heading back out as the resolved `>>> ...`
   command it names (see "Execution log": headings always record a
   command's canonical resolved `>>>` form, regardless of whether the
   triggering input was `>`, `>>`, or `>>>`). This is the command
   `continue` resumes -- never re-inferred, never re-confirmed: it was
   already resolved and confirmed once, when it first ran.
3. Reads every trace item already logged under that section, in order --
   each one's `[file:line]` link identifies exactly which pseudocode
   line it recorded having executed, and any `- **free-input:**` item
   identifies a completed `Agent.free(...)` call and its
   already-produced result.
4. Re-enters that command's own call chain from its first line, walking
   it exactly as a fresh invocation would (per `../README.md`'s
   "Execution model": one line at a time, local state only) -- but for
   each line already accounted for by step 3's recorded items, in the
   same order, treats it as already done: takes its previously-logged
   result (or, for a free-input line, its previously-logged `task`'s
   already-produced return value) as that line's outcome, without
   appending a new trace item and without re-performing its side effect
   a second time. This is what makes `continue` safe to run on a command
   that already had real effects (files written, Nodes purged) partway
   through -- it never replays a step that already happened.
5. The instant execution reaches a line that step 3's record doesn't
   already cover, `continue` stops treating anything as already-done and
   proceeds exactly like ordinary execution from there: each subsequent
   traced line gets its own new trace item appended to the *same*
   section (never a new `##` heading -- this is a continuation of that
   command's own section, not a new command), each `Agent.free(...)`
   call gets its free-input item logged before it runs, same as any
   command's first pass.
6. When the command's own steps run out, appends the `**Overview:**`
   paragraph closing the section -- same discipline as any command,
   reporting what the command as a whole actually produced (its full
   effect, not just the portion `continue` itself carried out).
7. Whatever the resumed command returns, or raises, is `continue`'s own
   return value/error -- reported back the same way running that command
   directly would have been.

### Example

Given `./log.md` ends with:

```text
## `>>> build Overview`

- [paths.py:52](agent/agent/paths.py#L52)
  ...
```

and no closing `**Overview:**` after it -- e.g. the agent's session was
interrupted mid-`build` --

```text
>>> continue
```

picks `build Overview` back up: skips re-running whatever step
`paths.py:52` already logged, and carries on from `build`'s next
pseudocode line, still under the same `` ## `>>> build Overview` ``
section, until `build` finishes and its `**Overview:**` is written.

Remark: `continue` only ever resumes the *single* most recent unfinished
section -- it doesn't scan further back, and it doesn't accept a target
command or heading to resume instead. If more than one section is ever
left unfinished (e.g. from repeated interruptions), only the last one is
reachable this way; the log's append-only, never-truncated nature means
every earlier one stays on record regardless, just not resumable through
this command.

Remark: because step 4 trusts step 3's recorded items as ground truth
for what already happened, `continue` is only as safe as that record is
complete -- consistent with `../README.md`'s requirement that every
traced line's entry be written *at the moment* it executes, never
batched afterward. A log built to that discipline is exactly what makes
`continue`'s replay-detection reliable; one that wasn't (a step that ran
but never got logged) would make `continue` skip a line it thinks
already happened, or worse, redo one that did.
