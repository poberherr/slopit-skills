---
name: slopit-review
description: Use when asked to review a PR in a repo indexed on hosted slopit (MCP), e.g. "/slopit:slopit-review 42" or "review PR 42" - reads the PR's change set (what the change touched and how it relates to the project) and links the board for the human.
---

# slopit-review: review a change from its change set, not from the raw diff

Target: `$ARGUMENTS` (a PR number or a PR url). Empty: ask which PR. A branch, sha or checkout path is not on hosted
slopit: ask for the PR number.

Every tool takes `repo` as `owner/repo`: derive it from `git remote get-url origin`; if that gives nothing usable,
call `mcp__slopit__repos` and ask. A tool answering `no access to ...` means the GitHub App is not installed on that
repo or GitHub denies you read access: point the user to https://slopit.bike-shed.io/docs/getting-started.

1. Call `mcp__slopit__review(repo, pr)`. It returns the report for the PR's change set. No change set yet: slopit
   builds it in the PR's CI run, so it appears after that run; there is no queueing and nothing to retry now. Say so
   and stop (`mcp__slopit__changes(repo)` lists the sets that are built).
   **Give the human the board link before any analysis**, so they can open it while you read:
   `https://slopit.bike-shed.io/r/<owner>/<repo>?change=pr-N` (behind GitHub login).
2. Read the result. It is JSON: the change graph (`nodes`, `edges`) plus a `report` object. Go through `report`
   top to bottom:
   - `files`, `generated`, `tests`, `added`, `modified`, `removed`, `ignored`: size of the change in graph terms.
     `generated` counts files of generated code (tagged, not drawn; the client operations they define appear under
     `clients`); `tests` lists the test files the PR touched (they are not part of the graph diff); `ignored` is
     the number of nodes outside the PR's own hunks.
   - `entries`: the routes / consumers / workflows that reach changed code (`id`, `label`, `reaches` = the changed
     nodes). These are the user-facing paths the change can break. For each: say what path it is and what this
     change does on it.
   - `clients`: generated client operations the change calls (`op`, `label`, `route`) and the chain of changed
     callers (`callers`).
   - `unreached`: changed functions no entry reaches. Dead code, a wiring gap, or generated code.
   - `hubs`: changed functions with many callers (`callers` count, callers not walked); a change there is wide.
   - `data`: tables / topics / queues touched (`label`) with `mode` (`r`, `w`, `rw`, `produce`, `start`); `via` is
     the witness path through unchanged helpers when a changed function does not hit the datum itself.
   - `guards`: `added` and `removed` locks, transactions, auth checks. A removed guard is a question for the author.
   - `funnels`: new functions many callers go through (`fanin`); review them first.
   - `removed_writers`: functions removed that wrote data (`tables`): nobody writes it any more after this change.
   - `unknowns`: edges slopit could not resolve inside changed code (`node`, `reason`); read that source by hand.
   - `services`: the services the change touches with `added` / `modified` / `removed` counts; the sibling key
     `crossing` is the number of edges that cross between services.
3. Tracing one entry in execution order and the package view are not on hosted slopit: read the code at the
   `file:line` the tools name. When `report.services` lists two or more, call
   `mcp__slopit__service(repo, name, height, change)` with `change` set to `pr-N` for each to see the surrounding
   picture; the crossing edges are the contracts to review.
4. Then read the diff (`gh pr diff <N>`, or `git diff <merge-base>..<head>`) only for the functions the report
   named: what to test, what looks risky, what is missing.
5. End with the board link again and a list of what in the report was noise or what you wanted and did not get.
   That list is what gets improved next.
