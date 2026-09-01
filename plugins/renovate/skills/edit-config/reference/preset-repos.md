# Shared preset repositories

A preset repo is a git repository whose JSON/JSON5 files Renovate fetches as
presets. A typical layout:

```
default.json            # what `github>org/repo` / `local>org/repo` resolves to
best-practices.json     # `github>org/repo:best-practices`  (":" = a file in the root)
system/registries.json  # `github>org/repo//system/registries`  ("//" = a path)
packages/internal.json  # a selector preset, extended from inside packageRules
renovate.json           # the repo's OWN config -- a consumer like any other
```

Everything below was checked against Renovate 44.x source and the debugger;
the version thresholds are exact, everything else should be re-resolved on the
consumer's pinned version before it is quoted.

## Reference syntax

| form                                                | resolves to                                                                                                                                                                                                                                                                                                                              |
| --------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `local>org/repo`, bare `org/repo`                   | `default.json` of that repo, fetched through the platform Renovate runs on (honours the configured `endpoint`). A missing `default.json` falls back to `renovate.json` with WARN `Fallback to renovate.json file as a preset is deprecated`.                                                                                             |
| `github>org/repo`, `gitlab>`, `gitea>`, `forgejo>`  | the same file on that host regardless of platform. `github>` hardcodes `https://api.github.com/` and ignores `endpoint`: on GitHub Enterprise only `local>` reaches the internal instance.                                                                                                                                             |
| `github>org/repo:file`                              | `file.json` in the repo root. `.json5` / `.jsonc` are kept when written; any other name gets `.json` appended (`:app.js` -> `app.js.json`).                                                                                                                                                                                             |
| `github>org/repo:file/sub`                          | key `sub` inside `file.json` (a sub-preset). At most three `/` segments are read; a fourth is dropped silently.                                                                                                                                                                                                                          |
| `github>org/repo//path/to/name`                     | `path/to/name.json` (or `.json5`/`.jsonc` when written). Cannot be combined with `:sub`: `Sub-presets cannot be combined with a custom path (...)`.                                                                                                                                                                                      |
| `...#v1.2.0`, `...#branch`, `...#refs/heads/main`   | any repo-hosted form at a tag, branch or full ref. Accepted and silently ignored on npm, http and internal presets (`config:recommended#v1` pins nothing).                                                                                                                                                                              |
| `...#v1.2.0(a, b)`                                  | parameters, always LAST. `{{arg0}}`, `{{arg1}}`, ... are replaced in the fetched body, `{{args}}` receives the raw string `a, b`. Anything after the closing `)` misparses silently on absolute references (`x(p1)#v1` -> params `p1)#v`).                                                                                             |
| `npm>name`, `@scope`, `@scope/name`                 | npm packages `renovate-config-name` / `@scope/renovate-config` / `@scope/name`. Deprecated upstream. Always `dist-tags.latest`: a `#tag` is folded into the package name and fails. `npm>owner/repo` (unscoped, with `/`) is a LOCAL repo. `npm>@scope/name` parses as npm only from 44.29.0; write the bare `@scope/name` on older runners. |
| `https://.../default.json`                          | the whole string is the URL; `#`, `//` and `:` are not parsed.                                                                                                                                                                                                                                                                          |
| `npm:name`                                          | **not a syntax** for npm packages. `npm:` is the bundled internal group (`npm:unpublishSafe`), like `config:`, `group:`, `packages:`.                                                                                                                                                                                                   |

`local>` is unsupported on the `codecommit`, `local` and `scm-manager`
platforms (`The platform you're using (X) does not support local presets.`).

`local>` versus `github>` is a trade-off, not a style choice: `local>` is
platform-neutral and follows `endpoint`; only `github>` / `gitlab>` / `gitea>`
pins get bump PRs (see "Tag pins and the `renovate-config` manager").

Since 44.29.0 a source prefix followed by `/`, `./` or `../` is a hard error:
`local>/org/repo`, `github>./org/repo` -> `Preset is invalid (...)` and the run
stops with `config-validation`. Older runners resolved them by accident. Fix
the string (`local>org/repo`); nothing is missing.

## Relative references inside the preset repo

Since Renovate 44.29.0 (renovatebot/renovate#45211) a file fetched as a preset
may reference sibling files relatively. Renovate rewrites the entry at fetch
time to `<source>><org/repo>//<path>#<parentTag>(<params>)`, so it inherits the
parent's **source, repository and tag** -- not the parent's parameters.

| write              | resolves to                                                                                                                                 |
| ------------------ | ------------------------------------------------------------------------------------------------------------------------------------------- |
| `/system/security` | `system/security.json` from the repo root -- the preferred form; it survives moving the referencing file                                    |
| `./security`       | relative to the directory of the referencing FILE (`presets/a.json` -> `presets/security.json`)                                             |
| `../security`      | the parent directory of the referencing file                                                                                                |
| `./app.json5`      | that exact file; a bare `./app` means `app.json`                                                                                            |
| `/group(eslint)`   | `github>org/repo//group#<tag>(eslint)`; own parameters survive, templated ones are allowed (`/group({{ env.TEAM }})`)                      |
| `/default`         | collapses to `github>org/repo#<tag>` with no `//default`, so it dedupes against the consumer's spelling; `//dir/default` stays as written  |
| `./{{arg0}}`       | works only because `{{argN}}` substitution runs before the rewrite; that is how a parent parameter reaches a child                          |

Rules that follow from the mechanism:

- **A relative reference names a preset FILE.** `./base/linters` fetches
  `base/linters.json`, never key `linters` of `base.json`; sub-preset keys need
  the `:file/sub` form. One preset per file.
- **No `#tag`, no `:sub`, no source prefix, no templated path.** `./x#v1`,
  `./x(p1)#v1`, `./x:sub`, `local>./x`, `github>../x`, a final `.`/`..`
  segment (`./..`, `./a/..`), `./a//b`, a trailing slash and unbalanced
  parentheses are all `Preset is invalid (...)` (validator:
  `extends: preset "./x#v1" is not valid`). The tag is always inherited and
  cannot be overridden. A path containing `{{` (`./{{ env.VARIANT }}/base`) is
  never rewritten and fails at resolution -- write the absolute
  `github>org/repo//{{ env.VARIANT }}/base` when the path must be dynamic.
- **Only under repo-hosted parents** (`github>`, `gitlab>`, `gitea>`,
  `forgejo>`, `local>`, bare `org/repo`). Inside an npm or `https://` preset
  the entry stays raw and every consumer fails.
- **Legal only inside a fetched preset.** In a repository's own config (top
  level, `packageRules[].extends`, update-type blocks), in an `inheritConfig`
  file and in `globalExtends` the run fails with
  `Relative preset reference cannot be resolved (<entry>). Relative presets can only be used within presets from a supported source, must stay inside their repository, and cannot be templated or used outside of a preset (for example in the repository config, inherited config, or globalExtends)`.
  `renovate-config-validator`, onboarding and reconfigure validation all
  accept such a file; only a run fails. rcd `run_config` reports it as
  `stageStatus.preset: "error"` while `accepted` stays `true`.
- **The rewrite walks the whole fetched body**: `packageRules[].extends`,
  update-type blocks, `ignorePresets` and `onboardingConfig`. A preset with
  `onboardingConfig: { extends: ["./base"] }` fetched at `#v3` writes
  `github>org/repo//base#v3` -- absolute and tag-frozen -- into every
  onboarded repo. Write the absolute string you want consumers to carry there.

Why convert: with absolute self-references (`github>org/repo//sub`, untagged)
a consumer pinned to `github>org/repo#v2.0.0` gets `default.json` at v2.0.0
and every nested file from the default branch, because each preset string is
fetched independently under a cache key equal to the raw string. Branch-testing
`#feature` tests only the root file. With relative references one consumer pin
or `#branch` covers the whole tree, and a fork resolves to itself.

### When a relative reference cannot be resolved

Canonicalisation never throws. An entry that escapes the repo (`../x` from a
root file), sits under an npm/http parent or is malformed is logged once in the
**preset author's** run as WARN `Could not resolve relative preset reference`
(`{ preset, parentPreset, err }`) and passed through unchanged. Every consumer
then fails with the message quoted above, suffixed
`. Note: this is a *nested* preset so please contact the preset author if you are unable to fix it yourself.`;
the repository worker logs `Repository has invalid config`, opens the
config-warning issue and creates zero updates -- including the valid parts of
the preset. A templated path fails the same way with no WARN at all.

Because the entry stays as written, a consumer can neutralise one broken entry
with `"ignorePresets": ["<entry exactly as written>"]` (`"../oops"`, not a
canonical form) and keep the rest of the parent. That escape hatch is the
reason the failure is deferred instead of failing the parent.

### Version floor and rollout order

- Runner **and** CI validator must be >= 44.29.0. Below it the runner parses
  `./system/registries` as a local repo named `./system/registries` and every
  consumer fails with `Cannot find preset's package (./system/registries)`; a
  validator below 44.29.0 reports `preset "./x" is not valid`.
- Order: bump the runner past 44.29.0 (read its pin in the runner's
  `package.json`, not `latest`), bump the validator pin, then convert.
- Upgrade hazard the other way: before 44.29.0 `./owner/repo` could
  accidentally resolve to `owner/repo` because the API URL parser dropped the
  `.` segment. Configs relying on that break on upgrade; write `owner/repo`.

### Converting a repo

Inside the preset files only (bare `extends` entries, including
`packageRules[].extends` and update-type blocks): `github>org/repo:file` ->
`/file`, `github>org/repo//path/name` -> `/path/name`. Leave `description`
strings that tell consumers what to extend in the full `github>org/repo...`
form -- consumers cannot use relative references -- and leave the repo's own
`renovate.json` on the absolute form. Then prove it as described under
"Testing a preset change": every node of `get_preset_tree` must carry the
branch or tag you resolved.

## Ignoring a preset the catalog pulls in

`ignorePresets` is a literal string compare against the **resolved** entry:
source prefix, `//path` and the inherited `#tag` included.

| consumer extends             | to drop `security/base.json` write                                                       |
| ---------------------------- | ---------------------------------------------------------------------------------------- |
| `github>org/repo#v2.0.0`     | `"ignorePresets": ["github>org/repo//security/base#v2.0.0"]`                             |
| `github>org/repo` (floating) | `"ignorePresets": ["github>org/repo//security/base"]`                                    |
| `org/repo` (bare)            | `"ignorePresets": ["local>org/repo//security/base"]` -- the prefix the user never typed |

`["./security/base"]`, `["/security/base"]`, the `:security/base` form and a
tagged consumer's entry without the tag never match, and the preset is merged
silently. A tagged entry stops matching when the root pin is bumped; re-pin it
with the root. Works at any nesting depth.

How to find the exact string: rcd `get_preset_tree` shows every node in its
canonical form; on a runner, `LOG_LEVEL=debug` and read `visitedPresets.merged`
in the record `Resolved shallow config, without merging internal presets`, or
the debug line `Ignoring preset <x>` once the entry matches.

Inside a preset, `ignorePresets` may be relative (rewritten like `extends`),
but the repository config's `ignorePresets` **replaces** the preset's own
whenever it is non-empty: a consumer that sets any `ignorePresets` switches
off every ignore the catalog set for itself. Catalogs must not rely on
self-ignores; consumers own opt-outs.

## One spelling per preset

Dedupe and the fetch cache compare raw strings. `org/repo`, `local>org/repo`,
`github>org/repo`, `github>org/repo//default`, `github>org/repo:base` and
`github>org/repo//base` are six strings for two files: each spelling is fetched
and merged once, and every mergeable array (`packageRules`, `addLabels`,
`description`, `postUpgradeTasks.commands`, `prBodyNotes`, ...) is concatenated
twice -- duplicate rules and doubled labels on PRs. The cycle guard
(`Already seen preset <x>` at debug) works per ancestor chain, so a preset
reached through two sibling paths is still merged twice; only a back-reference
into its own chain is cut.

Rules: consumers spell the root `github>org/repo#tag` or `local>org/repo`,
never bare `org/repo` or `//default`; nested presets reference the root only as
`/default` (which collapses to the consumer form); every internal reference in
one form; `:name` never dedupes against `//name`. Check rcd's
`treeSummary.duplicates`, which must be 0.

## Templated `extends`

Every `extends` entry is compiled as a Handlebars template with an empty
context before resolution (nested `packageRules[].extends` too). Available:
`env.*` (the allow-listed process variables such as `CI`, `HOME`, proxy vars;
plus `customEnvVariables` / `RENOVATE_ALLOWED_ENV`, or everything under
`exposeAllEnv`), `platform`, `renovateVersion`. Repo-config fields render
empty: `local>org/repo#{{ branchPrefix }}` becomes `local>org/repo#` and a
templated relative path becomes `.//base` -> `Preset is invalid (.//base)`, a
string nobody wrote. Conditionals work
(`{{#if env.CI}}github>a/b{{else}}github>c/d{{/if}}`).

- `customEnvVariables` and a repository's `env` block are **invisible to
  `globalExtends`**; only the real process environment is. Repository-config
  `extends` sees `customEnvVariables`.
- The validator skips syntax-checking any entry containing `{{`; the
  `renovate-config` manager skips templated entries, so a templated root is
  never pin-tracked.
- Documented use: branch-level preset testing with
  `local>org/presets#{{ env.GIT_REF }}` and `customEnvVariables: { GIT_REF: ... }`
  in the global config.

## Tag pins and the `renovate-config` manager

The built-in `renovate-config` manager (enabled by default on every Renovate
config filename except `package.json`) extracts top-level `extends` entries:

| entry                                   | extracted as                                                                                                                                       |
| --------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------- |
| `github>org/repo:best-practices#v1.1.0` | dependency `org/repo`, `currentValue: v1.1.0`, datasource `github-tags` -- bumped by a normal PR (`chore(deps): update dependency org/repo to v2`) |
| `gitlab>...#tag`, `gitea>...#tag`       | same on `gitlab-tags` / `gitea-tags`                                                                                                               |
| `github>org/repo` (no tag)              | `skipReason: unspecified-version` -- never pinned for you                                                                                          |
| `local>org/repo#v1.1.0`                 | `skipReason: unsupported-datasource` -- a `local>` pin is never updated                                                                            |
| `./x`, npm, http, templated entries     | skipped (`unsupported-datasource`; templated entries are not extracted at all)                                                                     |
| `extends` inside `packageRules`         | not tracked                                                                                                                                        |

(Verified with rcd `extract_deps` on `renovate.json`, Renovate 44.42.1.)

Consequences: "consumers pick a change up on their next run" is true only for
untagged consumers; tagged `github>` consumers receive a bump PR, and with
relative references one bump moves the whole tree. A "protective pin" is only
as protective as the review of that bump PR -- govern it with a rule on
`matchManagers: ["renovate-config"]`.

## Versioning a preset repo

- Version with git tags and Releases. One working shape: semantic-release with
  exactly `@semantic-release/commit-analyzer`, `release-notes-generator` and
  `github` (no `git`/`changelog`/`npm` plugins), so tags and Releases go through
  the API and a push-protected `main` does not block them; `tagFormat`
  `v${version}` continues from an existing `v1.0.0`.
- **Pin, then re-cut.** Before a breaking restructure: tag the current state,
  switch every consumer from floating `local>org/repo` to
  `github>org/repo#vX.Y.Z`, give teams a window, then ship the breaking change
  as the next major. Check what a tag points at with
  `gh api repos/org/repo/git/ref/tags/vX.Y.Z` and
  `gh api "repos/org/repo/contents/default.json?ref=vX.Y.Z"`. Unpinning later is
  each consumer's choice and their debt.
- Ship a compat root (`v1-compat.json`) that recreates the old bundle so a
  consumer migrates with one line; migrate its syntax so `--strict` passes but
  do not "fix" its behaviour -- that belongs in the new root. Before moving
  consumers between roots, `get_provenance` (no key) both roots and diff the
  option lists: anything only the old root set disappears for every migrated
  repo; fix it on both roots in the same preset PR.
- Every merged Renovate `fix(deps): ...` PR cuts a patch release. If that is
  churn, set `semanticCommitType: "chore"` on those deps with a rule. Put
  `":semanticCommits"` explicitly in the repo's own `extends`: the default
  `auto` flips in a mixed history and bot PRs arrive as
  `Update actions/setup-node action to v7`, failing any title lint and never
  releasing.
- On squash-merge repos set the squash title to the PR title (reported, not
  verified: with `COMMIT_OR_PR_TITLE` a single-commit PR squashes with the
  commit subject and a stray `BREAKING CHANGE:` line in a branch commit cuts an
  unintended major). Title lints must not cap header length -- grouped
  Renovate titles enumerate every dependency (see `validation.md`).

## Designing a catalog

- **One preset does one thing**: it finds dependencies (managers), or groups
  them, or automerges them, or schedules them, or decides trust -- never a mix.
  A rule with `automerge: true` and `schedule` in one preset is a smell:
  consumers who want "automerge always" cannot take one without the other.
  Cross-concern bundles are their own `features/` presets.
- **Roots carry no opinion**: the upstream base (`config:recommended` or
  `config:best-practices`) plus the non-optional `system/` presets
  (registries, security floor). Generic labels are set once in the roots
  (`addLabels: ["dependencies", "{{{updateType}}}"]`); presets add only
  purpose-specific labels. A taxonomy that worked: `system/`, `lang/` +
  `frameworks/` (managers only), `packages/`, `groups/`, `automerge/` (pure
  toggles in inverse pairs such as `non-breaking-external` /
  `breaking-external`), `schedules/`, `security/`, `features/`, `teams/`.
- **`packages/` selector presets**: a file with only `$schema`, `description`
  and `matchPackageNames`, mirroring Renovate's `packages:*`. Consumed as
  `packageRules: [{ "extends": ["/packages/internal"], "automerge": true }]`
  inside the catalog and `["github>org/repo//packages/internal"]` from
  consumers, so a `groups/` preset and a consumer's automerge or
  `postUpgradeTasks` rule share one definition and cannot drift. Names must
  match how the manager names the dependency (a GitHub slug for
  `github-tags`, `group:artifact` for Maven). The validator rejects a bare
  selector file outside a `packageRule`; CI wraps them (see `validation.md`).
- **Team overlays** are `teams/<team>.json` files in the shared repo, gated by
  a CODEOWNERS line after the default-owner line (a team named there must hold
  write access or the entry is silently ignored). They extend the recommended
  root plus opt-in presets and carry **deltas only**: nothing the chain already
  provides (generic labels, `automerge: false` on majors the automerge preset
  never grants, repeated `matchUpdateTypes` lists). Consumers extend the
  overlay **instead of** the root, never next to it (that merges the `system/`
  presets twice).
- **No single-package carve-outs in a shared or team preset.** A hold, a
  `pinDigests: false` for one action or an approval quirk for one package goes
  into the consuming repo's `renovate.json`; publish the block in the
  migration guide instead. A hold belongs there only when CI cannot catch the
  break; when a required check already goes red on the upgrade, add no hold
  and let the PR stay red. Never `ignoreDeps` for "cannot upgrade yet" -- it
  hides security updates too.
- **The file path is public API.** Renaming a preset file breaks every
  consumer that extends it; when a preset's meaning widens, reword its
  `description` and README row. Pair a rename only with a shim (old file
  extending the new one) and run
  `gh search code --owner <org> "<preset path>"` first.
- **Descriptions** state what the preset or rule is FOR; they surface in the
  dependency dashboard and PR bodies, not in your commit history. Where the
  justification and links live is an org style choice -- a shared repo may cap
  descriptions (purpose only, a length limit, CI-enforced) and keep reasoning
  in the README and PR body; the consumer's own config is where a long
  rationale goes. A wrapper preset (`description` + `extends` only) and a
  selector preset (`description` + `matchPackageNames` only) lose their
  description on resolution; use `overrideDescription` (44.41+) when the
  summary must survive, or put the rationale on the rule that extends the
  selector.
- `$schema: "https://docs.renovatebot.com/renovate-schema.json"` on every
  file: it is the only check for enum values and array element types.
- Prefer `matchSourceUrls` for "everything published from this monorepo",
  exact `matchPackageNames` when a scope mixes real packages with platform
  binaries. `groupName` supports templating.

## Testing a preset change

1. **With the debugger.** `run_config` a consumer config
   (`{"extends": ["github>org/repo#<branch>"]}`) before and after. Private
   repos need `RCD_GITHUB_TOKEN` / `GITHUB_TOKEN`. Read `stageStatus.preset`
   and `presetErrors` **before** `accepted`: a failed fetch yields
   `accepted: true` with the entry "expanded into 0 presets", and every
   provenance or simulate answer on that run describes an empty config. A
   token-less fetch of a private preset and a typo both read
   `Cannot find preset's package (...)` / `dep not found`, so a fetch failure
   is not evidence either way -- confirm the path with
   `gh api repos/org/repo/contents/<path>.json`. Then `get_preset_tree`: every
   nested node must carry `//<path>#<branch>`; `treeSummary.duplicates` must be
   0. Then `compare_simulations` the two runs against the dependencies the
   change targets plus one it must not touch.
2. **Without the debugger.** Global config file
   `export default { extends: ["github>org/repo#<tag>"], repositories: [] }`
   (`module.exports` where no ESM `package.json` governs the file), then
   `RENOVATE_TOKEN=$(gh auth token) RENOVATE_CONFIG_FILE=$PWD/cfg.js npx -y renovate@<pinned> --dry-run=lookup`.
   A resolvable reference proceeds to
   `WARN: No repositories found - did you want to run with flag --autodiscover?`;
   a bad one stops with `INFO: Throwing preset error` /
   `ERROR: config-presets-invalid`. Always run the bogus-tag control
   (`#v9.9.9`) alongside, or a pass is a silent no-op. More recipes in
   `validation.md`.
3. **In CI**, point consumers at the branch with
   `github>org/repo#{{ env.GIT_REF }}` (see "Templated `extends`").
4. **What does not test a branch.** A repository-level `--dry-run` reads
   `renovate.json` from the default branch, so it cannot test a PR-branch
   config (`RENOVATE_BASE_BRANCHES` does not change that). A green
   `renovate/reconfigure` branch check proves the config parses, resolves and
   validates **after migration** -- not that it is written in current syntax.
5. **Caching.** Presets are cached per run under `preset:<raw string>`; with
   the global `presetCachePersistence: true` they live 15 minutes in the
   package cache across repositories. "Pushed a change, next run did not see
   it" on such a runner is this; set `presetCachePersistence: false` in a local
   test config. A runner without the option refetches every run, so a stale
   preset there is not a cache problem.

## Rollout order

1. Land the preset PR first. A consumer that references a path or tag that is
   not on the default branch fails its **whole** config with
   `config-validation`, not just the new rule.
2. Consumer PRs stay draft until the preset is merged. To test a consumer
   against the branch meanwhile, reference it with an explicit `#<branch>`
   inside `packageRules[].extends` (or the templated root of step 3 above).
3. Bump the runner past a feature's version floor before the catalog uses the
   feature (relative refs 44.29.0, `overrideDescription` 44.41, depType
   `workflow` 44.43.0). Read the consumer's pinned Renovate version, not
   `latest`.
4. Internal presets bundled with Renovate (`group:*`, `packages:*`,
   `monorepo:*`) are the opposite: a change there ships with the next Renovate
   release only.

## Preset-repo CI

Every preset file is validated with the pinned validator, `--strict
--no-global`, discovered dynamically; the repo's own config is validated as a
repo config; `packages/` selectors are wrapped; same-repo
`packageRules[].extends` are inlined from the working tree; then a consumer
config is resolved at the pushed branch. The recipe, the validator's blind
spots and the exact messages are in `validation.md`; the error table and log
recipes are in `debugging.md`.
