---
name: slopit-explain
description: Use when asked how one service, route or function reaches another on a repo indexed on hosted slopit (MCP), e.g. "/slopit:slopit-explain checkout ledger" - walks the graph between the two ends, reads the code where the graph has no edge, and lists the missing edges.
---

# slopit-explain: how does A reach B? The graph is the map, the code fills the gaps

Target: `$ARGUMENTS`: `<A> <B>` or `<A> <B> "<question>"` (two ends: services, routes or functions). One end only:
see "One end" below. No end: ask. Use the slopit MCP tools (`mcp__slopit__search|slice|reach|service|services`). The
user may name the ends in either order; the answer is in runtime order. If a third service is the real start (the
caller of both), say so and start there.

Every tool takes `repo` as `owner/repo`: derive it from `git remote get-url origin`; if that gives nothing usable,
call `mcp__slopit__repos` and ask. A tool answering `no access to ...` means the GitHub App is not installed on that
repo or GitHub denies you read access: point the user to https://slopit.bike-shed.io/docs/getting-started.
Saved stories and the board canvas are not on hosted slopit: the answer is the text you print.

1. Resolve each end to a node id with `mcp__slopit__search(repo, q)` (pass the words; the hit shows type and file,
   `svc:<path>` for a service). Use the id for `slice`/`reach`/`service`; `service` also takes a bare service name.
   Several hits: list them and ask which; do not pick (headless: take the best-ranked hit and say so in the
   unresolved lines).
2. Map. `mcp__slopit__service(repo, B, height)` lists B's routes, tables and topics; `mcp__slopit__reach(repo,
   target)` on each route, table or topic gives the callers and handlers on the other side.
   `mcp__slopit__slice(repo, <A-id>, depth)` shows what A reaches; go to function height only where the two meet or
   stop, with a small `depth` (a central function prints thousands of lines). Callers and callees of one function
   are not a verb on hosted slopit: read the code at the `file:line` the tools name.
   If A and B already connect by edges, there is no boundary gap: the sequence is the path, go to 4.
   Otherwise the last reached function on A's side and the first on B's side are the frontier: the gap candidates.
3. Evidence. Declared wires and wire suggestions are not on hosted slopit. Read the code at the frontier
   functions, both ends, nothing else yet.
4. Classify every hop the graph lacks:
   - `GAP call`: same process, the extractor missed it (attribute call `self.x.m()`, callback, decorator dispatch).
   - `GAP wire`: crosses a system boundary (header, webhook, SDK event, queue, SaaS). Gets a pasteable
     `[[wire]]` block as text. One per integration boundary.
   For each: carrier or call form, direction, the two endpoint node ids, `file:line` on both sides.
   While reading a frontier function, read its branches, not only its calls: which condition arms this path
   (one hop upstream of A's start), which guards gate each hop, which hops run only on abort, timeout or fallback,
   and how the path is torn down. A hop that runs only on a branch is marked so; never present it as the happy path.
   A matching call name does not mean the branch fires: read the predicate inside the target too (a handler that
   filters on an identity, a guard that returns early when another side already owns the work). The reverse holds for
   abort paths: a probe sent on cancel can still return "committed" and complete the order; read what the
   probe's result does before marking the hop abort-only. Code that has routes and handlers but no live caller is
   staged, not live: name the config flag that gates it (both sides) and its default.
   Ownership / state questions ("who owns X at each step"): the carrier is usually one field or attribute, not a
   table. Find its writers (`mcp__slopit__reach(repo, <table>)` lists who writes it; when the only writer shown is
   a hub like `save()`, read the hub's callers in the code, then grep the field name for attribute and
   struct-field writes, which have no edges), then the readers whose branches gate an effect, and order them by
   runtime. The sequence in 5 then names the owner after each step.
   An end that looks outside the graph (a browser app) is often indexed: `mcp__slopit__search(repo, q)` for the
   other side's client before calling it unresolved.
5. Answer, about 60 lines: first the arming condition in one line, then one numbered sequence in runtime order,
   `N. <node> → <node>  [edge|wire|GAP call|GAP wire][, if <guard> | abort only | fallback]  <one line>`,
   then teardown in one or two lines. Then a section "Missing edges" listing each `GAP call` / `GAP wire` with both
   `file:line` and, for a `GAP wire`, the block as text (`from`/`to` = node id when `file:Name` is ambiguous, e.g. a
   method and a module function of the same name):
   ```
   [[wire]]
   from = "<node id | file:Name | path>"
   to = "<node id | file:Name | path>"
   via = "<carrier>"
   ```
6. Verify. For every step of the answer open each `file:line` you cite and confirm the identifier named in the hop
   and the predicate in the guard are there. The code does not show it: mark the step `GAP` or unresolved. Same
   identifier at another line: fix the line. Never delete a step to make the answer look verified.
7. Close with what you could not resolve and why, one line each: an ambiguous end, no code at the frontier, a
   carrier the code does not show, an end outside the graph (a browser app, a SaaS, an SDK event).

## One end

`$ARGUMENTS` with a single id (`<A>` or `<A> "<question>"`; the second token is a question when it holds a space or
is not an id): what feeds A, what it computes, what reads it. Steps 1 (A only), 6 and 7 apply.
- Map. A table or model: `mcp__slopit__reach(repo, <A-id>)` lists writers and readers. Anything else:
  `mcp__slopit__slice(repo, <A-id>, depth)` (callers and callees of a function are not a verb on hosted slopit: read
  the code at the `file:line` the tools name).
- Read the code that writes A (the model SQL, or the function) and the first reader; nothing wider.
- Answer in runtime order: inputs → A → outputs, each with `file:line`; a question is answered against that.

Do not:
- list a missing edge for a guess: both sides need a `file:line` that shows the hop.
- grep the whole repo before reading the frontier functions.
- claim a hop the code does not show; mark it GAP or unresolved.
- mark a step verified without reading its `file:line`.
