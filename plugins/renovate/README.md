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

| skill                       | what it does                                                                                                                                                                                                                              |
| --------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `/renovate:edit-config`     | Add, change, remove or migrate configuration: packageRules, grouping, automerge, schedules, `minimumReleaseAge`, ignores, custom managers, deprecated syntax, preset-repo restructuring. Resolves before, edits, resolves after, compares. |
| `/renovate:best-practices`  | Reviews a config against a checklist of practices with reasons, verifies each finding on the resolved config, and reports verdict-first with the exact edit.                                                                              |

Both load automatically when the conversation is about writing or reviewing a
Renovate config. Diagnosis without a change ("why did Renovate group these?")
is the debugger plugin's own `debug-renovate-config` skill.

## What the skills know

The reference files under `skills/*/reference/` were distilled from several
months of sessions maintaining Renovate, the debugger and shared preset repos.
Every claim was re-verified against Renovate 44.x source and through the
debugger; the version the debugger resolved with is part of every answer, and
the files say which thresholds are exact and what to re-resolve per pin.

- `edit-config/reference/matchers-and-rules.md` — merge order, matcher
  semantics, fail-closed inputs, update-type blocks, `group:` presets in
  rules, grouping, the options that mislead.
- `edit-config/reference/migrations.md` — what Renovate rewrites deprecated
  options into (and what it refuses instead).
- `edit-config/reference/preset-repos.md` — `local>` / `github>` / `//` / `:`
  / `#tag` / `(params)` syntax, relative references and their exact error,
  `ignorePresets` semantics, exact-string dedupe, templated `extends`, tag
  pins and the `renovate-config` manager, versioning a preset repo, designing
  a catalog, testing a preset change on a branch, rollout order.
- `edit-config/reference/custom-managers.md` — the preference order before
  writing a regex manager, RE2 constraints, proving extraction.
- `edit-config/reference/validation.md` — the canonical
  `renovate-config-validator` invocation, what it checks and its blind spots,
  exact messages, a CI recipe for a preset repository, the
  `renovate/reconfigure` branch, dry-run recipes, and the required-check rules
  for repos that consume Renovate PRs.
- `edit-config/reference/debugging.md` — `skipReason` before anything else,
  debug-log recipes, the preset error message table, preset caching, rcd
  pitfalls, Renovate's commit statuses and artifact retries, why a PR did not
  automerge, `renovate/*` branch semantics, self-hosted runner notes, gating
  on the consumer's pinned version.
- `best-practices/reference/checklist.md` — practices with reasons, and an
  anti-pattern table.
- `best-practices/reference/presets.md` — what `config:recommended`,
  `config:best-practices`, `helpers:pinGitHubActionDigestsToSemver` and the
  `customManagers:*` presets actually contain, and how to re-resolve them.
- `best-practices/reference/automerge-gates.md` — what `automerge: true` does
  and does not do: the platform-side blockers (required reviews and CODEOWNERS,
  required signatures, `allow_auto_merge`, aggregator jobs, push-only
  workflows, merge queues), which checks to require and which never to.
- `best-practices/reference/github-actions.md` — SHA pins with the exact
  released version in the comment, `# main` versus `# vX.Y.Z`, bare SHAs that
  Renovate cannot see, reusable workflows that must stay on tag refs.
- `best-practices/reference/report-format.md` — how to write findings a
  reader can act on.

## Without the MCP server

The skills fall back to the same engine on the command line:

```bash
npx -y @renovate-config-debugger/cli validate renovate.json --format json
npx -y @renovate-config-debugger/cli compare before.json after.json --dep '{"depName":"react","updateType":"major"}'
```

Renovate's own validator is a separate, narrower gate — it never fetches
top-level `extends` and accepts relative references in any file (see
`edit-config/reference/validation.md`). Invoke it with the bot's pinned
version, as repo config, in strict mode:

```bash
npx -y -p renovate@<the bot's pinned version> renovate-config-validator --strict --no-global renovate.json
```

Renovate 44.x requires Node 24 (`engines ^24.11.0`); on another Node major the
validator falls back from RE2 to JavaScript regexes with a warning and
`renovate` itself refuses to start.

## License

MIT. Renovate itself and the debugger are both AGPL-3.0-only.
