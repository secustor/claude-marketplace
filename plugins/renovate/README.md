# renovate

Claude Code plugin for **editing and reviewing Renovate configurations** —
`renovate.json` / `renovate.json5` / `.renovaterc`, the `renovate` key of
`package.json`, shared preset repositories and custom managers.

It depends on [`renovate-config-debugger`](https://github.com/secustor/renovate-config-debugger)
(installed automatically from the same marketplace), whose MCP tools resolve a
config with Renovate's own code. Every edit these skills make is resolved
before and after, and proven with a simulated dependency update, instead of
being argued from the diff.

```
/plugin marketplace add secustor/claude-marketplace
/plugin install renovate@secustor
```

## Skills

| skill                       | what it does                                                                                                                                                                             |
| --------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `/renovate:edit-config`     | Add, change, remove or migrate configuration: packageRules, grouping, automerge, schedules, `minimumReleaseAge`, ignores, custom managers, deprecated syntax, preset-repo restructuring. Resolves before, edits, resolves after, compares. |
| `/renovate:best-practices`  | Reviews a config against a checklist of practices with reasons, verifies each finding on the resolved config, and reports verdict-first with the exact edit.                              |

Both load automatically when the conversation is about writing or reviewing a
Renovate config. Diagnosis without a change ("why did Renovate group these?")
is the debugger plugin's own `debug-renovate-config` skill.

## What the skills know

The reference files under `skills/*/reference/` were distilled from several
months of sessions maintaining Renovate, the debugger and shared preset repos,
and every claim was re-verified against Renovate 44.42.1 through the debugger:

- `edit-config/reference/matchers-and-rules.md` — merge order, matcher
  semantics, fail-closed inputs, update-type blocks, `group:` presets in
  rules, grouping, the options that mislead.
- `edit-config/reference/migrations.md` — what Renovate rewrites deprecated
  options into (and what it refuses instead).
- `edit-config/reference/preset-repos.md` — `local>` / `github>` / `//` / `:`
  syntax, relative references, testing a preset change on a branch.
- `edit-config/reference/custom-managers.md` — the preference order before
  writing a regex manager, RE2 constraints, proving extraction.
- `best-practices/reference/checklist.md` — practices with reasons, and an
  anti-pattern table.
- `best-practices/reference/presets.md` — what `config:recommended`,
  `config:best-practices`, `helpers:pinGitHubActionDigestsToSemver` and the
  `customManagers:*` presets actually contain.
- `best-practices/reference/report-format.md` — how to write findings a
  reader can act on.

## Without the MCP server

The skills fall back to the same engine on the command line:

```bash
npx -y @renovate-config-debugger/cli validate renovate.json --format json
npx -y @renovate-config-debugger/cli compare before.json after.json --dep '{"depName":"react","updateType":"major"}'
```

## License

MIT. Renovate itself is AGPL-3.0; the debugger is AGPL-3.0-only.
