# slopit skills

slopit builds a code graph of your repository (routes, tables, functions, who calls what) and serves it to Claude
over MCP. These skills teach Claude to plan from the data flow, review a PR from its change set and explain how one
service reaches another.

## Install

```sh
claude mcp add --transport http --scope user slopit https://slopit.bike-shed.io/mcp
```

Then `/mcp` -> slopit -> Authenticate (GitHub login). Then the skills:

```sh
claude plugin marketplace add poberherr/slopit-skills
claude plugin install slopit@slopit-skills
```

## Skills

- `slopit`: plan from the data flow of a route, table or function. Example: `/slopit:slopit POST /documents/upload`
- `slopit-review`: review a PR from its change set. Example: `/slopit:slopit-review 42`
- `slopit-explain`: how does A reach B. Example: `/slopit:slopit-explain billing ledger`

Getting started: https://slopit.bike-shed.io/docs/getting-started

Examples you can open without an account: https://slopit.bike-shed.io/examples

## License

MIT for the files in this repository.
