# `cd`

## Syntax

```text
>>> cd <path>
```

- `<path>` — a relative path (POSIX-style, `.`/`..` segments allowed)
  from `agent.cwd` (see `../agent.md`) to another folder under
  `./Topology`: a plain organizational folder or a Node, either is a
  valid target — `cd` doesn't care which, it only moves where
  `agent.cwd` points. Examples: `Group`, `Sub1/Sub2`, `..`, `../Other`,
  `.` (a no-op).

## Behavior

1. Joins `<path>` onto `agent.cwd`, resolving `..`/`.` segments the same
   way a shell does. Every segment along the way must already exist as a
   folder under `./Topology` — `cd` never creates anything. An error if
   any segment doesn't exist, if the result would move above
   `./Topology` itself, or if any segment is (or is nested inside) a
   `.class` directory — `.class/` is reserved (see `../Node/nesting.md`)
   and isn't part of the addressable Node/organizational-folder tree
   `agent.cwd` moves around in.
2. On success, sets `agent.cwd` (see `../agent.md`) to the resolved path.
3. Reports back the resolved path (e.g. `Group/Sub`, or `.` at the root).

### Example

```text
>>> cd Group
```

moves `agent.cwd` to `Group`. From there:

```text
>>> cd Sub1/Sub2
```

moves it to `Group/Sub1/Sub2`. Then:

```text
>>> cd ..
```

returns it to `Group/Sub1`, and:

```text
>>> cd ../..
```

returns it all the way to `.`, the root.

## Purpose

`agent.cwd` is what every other command's `<NodePath>`/`<TransformPath>`
argument resolves relative to (see `../agent.md`) — `do`'s raw Python
line included, via `node(path)`/`transform(path)` (see
`../Node/nesting.md`). `cd` is the only thing that changes it, the same
way `cd` works in an ordinary shell: it doesn't touch anything under
`./Topology` itself, it just changes what a subsequent relative path
means.
