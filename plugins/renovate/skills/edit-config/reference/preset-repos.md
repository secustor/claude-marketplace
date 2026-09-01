# Shared preset repositories

A preset repo is a git repository whose JSON/JSON5 files are fetched by
Renovate as presets. The user's own layout (`secustor/renovate-config`,
`secustor/backstage-renovate-config`) is the reference shape:

```
default.json          # what `github>org/repo` / `local>org/repo` resolves to
app.json5             # `github>org/repo:app.json5`  (":" = a file in the root)
presets/no-esm.json   # `github>org/repo//presets/no-esm`  ("//" = a path)
renovate.json5        # the repo's OWN config — a consumer like any other
```

## Reference syntax

| form                                  | resolves to                                                   |
| ------------------------------------- | ------------------------------------------------------------- |
| `local>org/repo`                      | `default.json` of that repo on the platform Renovate runs on  |
| `github>org/repo`, `gitlab>org/repo`  | same, on that host regardless of platform                     |
| `github>org/repo:file`                | `file.json` in the repo root; keep the extension for `.json5` (`:app.json5`) |
| `github>org/repo//path/to/preset`     | `path/to/preset.json` (or `.json5` when written)              |
| `github>org/repo#v1.2`                | any of the above at a tag or branch                           |
| `preset(arg)`                         | `{{arg0}}` in the preset body is replaced                     |
| `npm:name`, `http://…`                | not repo-hosted; no auth; no relative references inside       |

The platform-neutral `local>` form is right for a self-hosted or Mend-hosted
consumer on the same platform as the preset repo; `github>` pins the host.

## Relative references inside the preset repo

Since Renovate 44.29 a preset file may reference siblings as `./x`, `../x` or
`/x` (root of the repo). They inherit the parent preset's source, repo, tag
and parameters, so:

- `github>org/repo#feature` resolves the **whole chain** from `feature`.
  With absolute self-references (`github>org/repo//presets/x`) the nested
  presets silently fall back to the default branch, and branch-testing a
  preset repo tests nothing. This is the reason to convert.
- The extension is kept when written (`./app.json5`); a bare `./app` is
  `app.json`.
- They work only under a repo-hosted parent (`github>`, `gitlab>`, `gitea>`,
  `forgejo>`, `local>`), never under `npm:` or `http`.
- A relative reference at the top level of a repo's own config is refused
  (`PRESET_RELATIVE_NO_PARENT`); one that escapes the repo (`../../x`) is
  refused (`PRESET_RELATIVE_OUTSIDE_REPO`). The preset repo's own
  `renovate.json` therefore keeps an absolute reference.
- Parameters survive: `./tpl(weekly)` still renders `{{arg0}}`.

Converting a repo: change every `github>own/repo:file` to `./file` and every
`github>own/repo//path` to `./path` inside the preset files only; leave the
repo's own `renovate.json` alone; then prove it by resolving a consumer config
that extends the repo at `main` and one that extends it at the pushed branch,
and comparing a representative dependency across the two runs.

## Writing a preset that consumers can live with

- Put `$schema: "https://docs.renovatebot.com/renovate-schema.json"` at the
  top of every file; editors validate against it.
- Every `packageRules` entry carries a `description` (string or array) with
  the issue link that justifies it. A consumer reading their dependency
  dashboard sees those descriptions, not your commit history.
- A collection preset whose body is only `description` + `extends` loses its
  own description on resolution; add `overrideDescription` (44.41+) when the
  summary must survive.
- Prefer `matchSourceUrls` for "everything published from this monorepo"
  (`https://github.com/backstage/backstage`), `matchPackageNames` with exact
  names when a scope mixes real packages with platform binaries.
- Group by upstream with a template: `groupName: "Backstage plugin {{ lookup
  (split packageName '-') 1 }}"` is legal, `groupName` supports templating.
- `allowedVersions: "<5.0.0"` plus a description with the blocking issue is
  the documented way to hold a major back; delete the rule when the issue
  closes. Prefer it over `ignoreDeps`, which also hides security updates.
- The same preset reached twice through two paths is resolved twice; watch the
  digest's `duplicates` count.

## Testing a preset change

1. `run_config` a consumer config (`{"extends": ["github>org/repo#branch"]}`)
   before and after the change. Private repos need `RCD_GITHUB_TOKEN` /
   `GITHUB_TOKEN` in the environment; a failed fetch appears as a failed node
   and everything under it is missing, so check `presetErrors` first.
2. `compare_simulations` the two runs against the dependencies the change
   targets, plus one it must not touch.
3. Only then open the PR. Consumers pick the change up on their next run; there
   is no release step. Internal presets bundled with Renovate (`group:*`,
   `packages:*`, `monorepo:*`) are the opposite: a change there ships with the
   next Renovate release only.
