---
name: slopit
description: Use when planning a feature or changing a route, table, column, daemon or MCP tool in a repo indexed on hosted slopit (MCP) - shows the data flow (entry, Depends, calls, adapters, column writes/reads, unknowns) from the enriched code graph before you read source.
---

# slopit: plan from the data flow, not from grep

Hosted slopit (MCP) serves the code graph of a repo through the `slopit` MCP server. Every tool takes `repo` as
`owner/repo`: derive it from `git remote get-url origin`; if that gives nothing usable, call `mcp__slopit__repos`
and ask which one. A tool answering `no access to ...` means the GitHub App is not installed on that repo or GitHub
denies you read access: point the user to https://slopit.bike-shed.io/docs/getting-started.

## Before planning or editing

1. Find the start node. If the user names it loosely, `mcp__slopit__search(repo, q)` lists candidates with type and
   file.
2. Show the flow: `mcp__slopit__slice(repo, entry, depth)` for a route (`POST /x`), `daemon:<name>`, `mcp:<tool>` or
   `fn:<module.func>`; `mcp__slopit__reach(repo, target)` for a `<table>.<column>`: what reaches it, the callers and
   data that flow into it. Columns are folded into tables by default.
   A `service`-typed hit (`svc:<dir>`) is a whole service: `mcp__slopit__service(repo, name, height)` (entries in,
   data/topics/routes out, uses/used_by); `mcp__slopit__services(repo)` lists the catalogue. A repo whose slopit config
   declares no services shows none.
3. Unclear node, or the connection between two things: not on hosted slopit (no `explain` or `path` verb); read the
   code at the `file:line` the tools name, or see the `slopit-explain` skill for two ends.
4. The board, proposals, stories and re-indexing are not on hosted slopit: the index is built by the repo's CI run
   and is as fresh as its last push.

## Change sets: what a PR changed and how it relates to the project

Reviewing a PR? Use the `slopit-review` skill (`mcp__slopit__review(repo, pr)`). `mcp__slopit__changes(repo)` lists
the change sets that are built. A PR that is not built yet is built by the repo's CI run and appears after that run.
`slice`, `reach` and `service` also take `change` (`pr-<N>`) to read the graph as the PR leaves it. The board for a
change is `https://slopit.bike-shed.io/r/<owner>/<repo>?change=pr-N` (behind GitHub login).

## Reading the output

`mcp__slopit__slice` returns JSON: `entry` (the resolved start node id), `nodes` (every node reached, each with
`id`, `label`, `type`, `source_file` and `source_location`, i.e. the `file:line` to read, and `metadata`), `edges`
(`source`, `target`, `relation` such as `calls`, `depends_on`, `reads_table`, `writes_table`, `handled_by`,
`confidence`) and `warnings`. Walk the edges from `entry` by relation to build the flow: route -> depends ->
calls -> adapters -> writes and reads. `confidence` `EXTRACTED` is read from the framework; `INFERRED` is a guess;
a node carrying an `unknowns` list in its `metadata` is a risk to name. `depth` bounds the walk. Plain text
instead of JSON (e.g. `no access to ...`, `no graph yet`) is an error message: relay it.

## Rules

- Read source ONLY for nodes the slice names; cite them as `file:line`.
- Treat every Unknown and Warning as a risk: list it in your plan, never drop it silently.
- Propose the spec and edge cases in slice order: entry -> depends -> writes -> adapters.
- The state view, schema diagrams, wires, proposals and local extractors are not on hosted slopit: read the code at
  the `file:line` the tools name.
