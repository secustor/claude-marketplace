# Checklist — practices, their reasons, and the anti-patterns

Each item states the practice, why, and how to verify it on the resolved
config. Severity is a default; the repo's context can move it. Items marked
"repo settings" cannot be verified from the config: report them as
hypotheses with the `gh` recipe that would settle them
(`reference/automerge-gates.md`, `reference/report-format.md`).

## Baseline

- **`$schema` on every config and preset file.** Editors validate keys and
  values against it, and it is the **only** check for enum values and array
  element types: `renovate-config-validator` accepts
  `matchUpdateTypes: ["bogusvalue"]` and `labels: [1, true]`. (low)
- **`extends: ["config:recommended"]` at minimum; `config:best-practices`
  where digest-pinning PRs are acceptable.** `config:recommended` is the
  dashboard, sensible ignores, semantic commits and ~700 grouping rules;
  `config:best-practices` adds digest pinning for docker images and GitHub
  Actions (reusable-workflow calls included), `minimumReleaseAge` for npm,
  pinned devDependencies, weekly lock file maintenance, a 1-year abandonment
  flag and config self-migration (`reference/presets.md`). Before recommending
  it, check whether any `uses:` targets a reusable workflow whose OIDC trust
  is keyed on `job_workflow_ref`; if so the `pinDigests: false` carve-out goes
  into the same change (`reference/github-actions.md`). `config:base` is a
  migration alias only. (high) Verify: `get_preset_tree` shows the entry; the
  digest names the count.
- **One shared preset repo per org (named `renovate-config`, so onboarding
  finds it); every repo extends it; consumer files are `extends`-only.**
  Repo-local `packageRules` exist only for a repo-specific reason (a hold, a
  `postUpgradeTasks` recipe, a codegen exclusion). A fix in the preset reaches
  untagged consumers on their next run; consumers pinned to
  `github>org/repo#vX` receive a bump PR from the `renovate-config` manager
  (never for `local>` pins), so a breaking preset change is tag-then-pin
  (`../../edit-config/reference/preset-repos.md`). (medium) Verify:
  `get_provenance` — a repo-level option the preset also sets is a finding
  against the repo; an `extends` entry the chain already reaches is a finding
  (`treeSummary.duplicates` drops when it goes; the option's `writtenBy` stays
  the preset).
- **One spelling per preset.** Dedupe is exact-string: `org/repo`,
  `local>org/repo`, `github>org/repo`, `github>org/repo//default`, `:base`
  and `//base` are different presets to Renovate, and a file reached under two
  spellings concatenates its `packageRules`, `addLabels` and every other
  mergeable array twice. A team overlay is a replacement root — the consumer
  extends only the overlay, never the overlay next to the org root. (medium)
  Verify: `treeSummary.duplicates` is 0 or explained (a structural duplicate
  such as `helpers:pinGitHubActionDigestsToSemver` next to
  `config:best-practices` is harmless).
- **No deprecated syntax.** Renovate migrates silently and logs it at debug
  level in a run. It shows in the debugger's digest ("rewrote N deprecated
  options"), as `WARN: Config migration necessary` plus `Config migration
  diff:` in any `renovate-config-validator` run (exit 1 only with `--strict`),
  and never on a green `renovate/reconfigure` check (migration runs before
  validation there). Write exactly the migrated form Renovate produces
  (`../../edit-config/reference/migrations.md`); `:configMigration` (in
  `config:best-practices`) makes Renovate open that PR itself. (medium)
- **The config is accepted with no warnings.** A warning is a config that
  works by accident: `group:` presets in a rule, a selectors-only rule, `*`
  mixed with other patterns, global-only options in a repo file (`force` is
  applied anyway). The CLI reports global-only options only when validating
  as repo config:
  `npx -y -p renovate@<the bot's pinned version> renovate-config-validator --strict --no-global <file>`
  (a positional file is otherwise validated as global config, the permissive
  superset). It never fetches top-level `extends`, so a green run proves
  nothing about a remote preset, path or tag — `run_config` and
  `presetErrors` do (`../../edit-config/reference/validation.md`). (high)
- **CI runs the validator pinned to the bot's Renovate version**, under a
  `# renovate: datasource=npm depName=renovate` marker so the pin moves
  through the same gates as every other dependency. `@latest` validates
  against a version production does not run and pulls an unvetted release
  straight past the release-age floor. Read the pin from the runner's
  `package.json`; bump it in the change that moves the runner. (medium)
- **One dependency bot per repo.** A `.github/dependabot.yml` next to Renovate
  produces duplicate PRs. Before removing one kept "as a security backstop",
  confirm `vulnerabilityAlerts` / `osvVulnerabilityAlerts` in the resolved
  config (`get_provenance`) and the App's Dependabot-alerts permission. (low)

## Supply chain

- **GitHub Actions: full SHA plus the exact released version as comment, kept
  in sync by `helpers:pinGitHubActionDigestsToSemver`.** The comment is what
  **Renovate** reads: for a SHA ref `currentValue` comes only from it.
  `@<sha>` with no comment or a prose comment is
  `skipReason: unversioned-reference` — unmanaged, absent from the dashboard,
  unrescuable by any `packageRules`; `# main` tracks the branch head
  (`github-digest`); `# v7` tracks at major granularity; `# v7.1.1` is the
  form. Plain `helpers:pinGitHubActionDigests` (inside `config:best-practices`)
  pins but lets the comment rot. Where the platform does not enforce SHA
  pinning, this preset is the control — do not weaken it per repo. (high)
  Semantics, the two accepted comment shapes and the bare-SHA remediation
  order: `reference/github-actions.md`.
- **Exempt OIDC-gated reusable workflows from digest pinning.**
  `helpers:pinGitHubActionDigests` also rewrites
  `uses: owner/repo/.github/workflows/x.yml@v1.2.0` to a SHA; a trust policy
  keyed on `job_workflow_ref` (`refs/heads/main` / `refs/tags/v*`) then
  denies the caller (`Not authorized to perform sts:AssumeRoleWithWebIdentity`).
  Exempt the hosting repos with
  `{ "matchManagers": ["github-actions"], "matchPackageNames": ["Owner/repo"], "pinDigests": false }`
  (or `matchDepTypes: ["workflow"]` on Renovate >= 44.43.0), merge it before
  reverting existing pins (`pinDigests: false` stops future pins only), and
  keep callers on protected release tags. (high) Verify: `simulate` with
  `updateType: "pinDigest"` → `pinDigests: false`; a third-party action → `true`.
- **Pin docker image digests (`docker:pinDigests`)** for images built or
  deployed from the repo. Fixture and example files should not be named like
  a manifest at all (`Dockerfile.fixture` still matches `*Dockerfile*`;
  `container.fixture` does not). (medium)
- **A release-age floor in one non-optional preset; first-party exemptions as
  separate `minimumReleaseAge: null` rules.** `security:minimumReleaseAgeNpm`
  (in `config:best-practices`) is the built-in 3-day npm gate; an org floor
  belongs in a `system/`-style preset every root imports, so no consumer drops
  below it. Exempt what the org publishes in **separate** rules —
  `matchSourceUrls: ["https://github.com/<org>/**"]`, `matchRegistryUrls` for
  org-owned registries, namespace `matchPackageNames: ["@<scope>{/,}**"]` —
  because `match*` fields inside one rule are ANDed. The org's own actions
  need the exemption too, scoped as `matchManagers: ["github-actions"]` +
  `matchPackageNames: ["<Org>/**"]`: that manager names packages
  `owner/repo`, never a URL, so a URL inside `matchPackageNames` silently
  never matches (it held official `actions/*` for the full quarantine in one
  org's first preset). A shorter lane for trusted publishers is its own
  preset via `matchSourceUrls`, not a per-package carve-out. (high) Verify:
  `get_provenance minimumReleaseAge`; `simulate` a third-party npm dep, a
  first-party dep with `sourceUrl` set, and a first-party action — rules
  keyed on `matchSourceUrls` show up in `missingInputs` when the dep has no
  `sourceUrl`.
- **`vulnerabilityAlerts.minimumReleaseAge: "1 day"`.** The
  `vulnerabilityAlerts` default is `minimumReleaseAge: null` (verified with
  `get_option_docs`): vulnerability PRs bypass the floor entirely, so a
  fabricated advisory could rush a malicious "fix" release through. Set
  `vulnerabilityAlerts: { minimumReleaseAge: "1 day", addLabels: ["vulnerability"] }`
  next to `osvVulnerabilityAlerts: true` and
  `dependencyDashboardOSVVulnerabilitySummary: "all"`; the label makes the
  path checkable on a real PR (`... [security]` on
  `renovate/<datasource>-<dep>-vulnerability`, raised even while
  `prHourlyLimit` holds other updates). Needs the Dependabot-alerts permission
  on the App. (high)
- **The package manager's cooldown is an independent second gate.** pnpm >= 11
  (default on), npm `--before` and poetry quarantine at install time with
  **their** configured age: pnpm's `minimumReleaseAge` is in minutes
  (`10080` = 7 days), Renovate's a duration string. A lockfile that clears
  Renovate's gate can still fail `pnpm install --frozen-lockfile` with
  `Lockfile failed supply-chain policy check` /
  `ERR_PNPM_MINIMUM_RELEASE_AGE_VIOLATION`; automerged inside a group it leaves
  `main` red for every later Renovate branch. Reconcile them (Renovate's age
  at least the package manager's, or `minimumReleaseAgeExclude` mirroring
  Renovate's exemptions, scoped to your own publishing scopes);
  `minimumReleaseAgeBuffer` (44.x) covers companion packages published minutes
  apart. Keep all three layers: Renovate's gate (PRs), the package manager's
  (every install), frozen-lockfile builds with scripts disabled. (high)
- **No runtime "download latest, fail if older than N days" scripts, no
  `npx <tool>` in CI.** The age script gates on the publisher's cadence and can
  only reject the best release available (observed: every build failing for
  34 days because the newest runner release was 34 days old). `npx <pkg>`
  resolves a whole tree from the registry at run time, outside the lockfile
  and both release-age gates. Replace with a pinned `_VERSION` constant +
  `# renovate:` marker + `customManagers:githubActionsVersions` (an automerge
  rule then advances it), or an exact devDependency with a committed lockfile
  and `npm ci` / `pnpm exec`. (medium)
- **Lock file maintenance on, weekly, automerged only behind a required
  frozen-lockfile check.** `lockFileMaintenance: { enabled: true, automerge: true }`
  at the top level composes with `:maintainLockFilesWeekly`'s schedule. A
  lock that only evolves keeps stale transitive pins: one repo measured 573
  packages / 5 advisories evolved against 558 / 2 from a fresh resolve of the
  same manifest — disabling it in a shared preset costs security. Self-hosted:
  refreshing `mise.lock` needs the global `allowedUnsafeExecutions: ["mise"]`,
  otherwise a committed lock is never refreshed. (medium)
- **Holds: let a required check make the PR red; hold only what CI cannot
  catch.** When a required check fails the breaking upgrade, add no
  `allowedVersions`, no `matchCurrentVersion` guard, no manifest pin: the red
  PR is self-documenting, self-resolving (it merges when upstream unblocks,
  with no config cleanup) and visible on the dashboard, while a hold hides the
  upgrade and depends on someone remembering to lift it. Hold with
  `allowedVersions: "<10"` and a `description` (upstream link, lift condition)
  only when the failing PR would carry no information — a silent failure mode
  CI does not exercise. A CI version assertion or a README bullet is not a
  hold: Renovate re-proposes the major on the next run. Scope a hold to one
  workspace file with `matchFileNames` where needed. Never `ignoreDeps` for
  "cannot upgrade yet" (it hides security updates); `enabled: false` with the
  constraints and a revisit condition in `description` is the shape for
  structurally impossible upgrades and for codegen-output manifests. (medium)
  Verify: `simulate` the major — raised under a required check, filtered under
  a hold.
- **npm `overrides`: one top-level rule per package, exact, retired when
  redundant.** Two rules for one name (a nested plus a top-level one) fail npm
  inside Renovate's artifact step with `EOVERRIDE ... conflicts with direct
  dependency`; Renovate then commits `package.json` without the regenerated
  lockfile and `npm ci` on `main` fails (`lock file's X@a does not satisfy
  X@b`). Prefer lock file maintenance over an override as a CVE fix. Before
  keeping one, check it is reachable in the lock and that removing it
  reintroduces an advisory (`npm audit --json` diffed against a fresh
  `--package-lock-only` resolve); if it must stay, an exact pin lets Renovate
  govern the version instead of npm floating at each regeneration. (medium)
- **Judge action and package freshness by the last release**, not `pushed_at`
  or bot merges: JS actions ship a `dist/` built at release time, so consumers
  run dependencies as old as the last release. `abandonments:recommended`
  flags one year. (low)

## Automerge

Every item here has its long form, the `gh` audit recipes and the triage
order in `reference/automerge-gates.md`. Repo-settings items are hypotheses
until those recipes have been run.

- **`automerge: true` only arms platform auto-merge.** `allow_auto_merge`
  off, a required review with no bypass actor, `require_code_owner_review`
  plus a catch-all CODEOWNERS line, `required_signatures`, a required context
  nobody reports, and a merge queue whose workflows lack `merge_group` each
  block it independently. When Renovate merges as the bot instead
  (`platformAutomerge: false`) onto a merge-queue base, a bypass actor set
  to `always` rather than `exempt` blocks the enqueue on every code-owned
  path, silently. A non-required failing check does **not**: red PRs
  merge, and `renovate/artifacts` is not a gate. Say "configured" or
  "actually merges". (high, repo settings)
- **Automerge only what a required check tests, on the PR.** The required
  workflow must run on `pull_request`/`merge_group` — push-only `deploy.yml`
  / `publish.yml` see the change after the merge (an org audit found 140 of
  177 post-merge breakages in push-only workflows) — must actually invoke the
  test runner (`grep -nE 'pnpm test|npm test|vitest|jest|ava|cargo test|pytest'`
  on the required workflow; a test-runner major merged green where the check
  ran only install, typecheck and synth), and must include a frozen-lockfile
  install (`npm ci`, `pnpm install --frozen-lockfile`, `cargo --locked`).
  (high, repo settings)
- **Required checks: pinned to the reporting app, aggregated by one terminal
  job.** `integration_id: 15368` for GitHub Actions (an `app_id: null` context
  is greenable by any write actor); reusable-workflow checks are composite
  `<caller> / <called>` names; never `renovate/stability-days` or
  `renovate/artifacts`. The aggregator `needs:` every job in the **same**
  file, runs `if: always()` and fails on anything but `success` — `skipped`
  propagates from a failed upstream and passes a
  `!contains(needs.*.result, 'failure')` gate. Flip protection to it only after
  it reported once on the default branch. (high, repo settings)
- **CODEOWNERS carve-out for the paths Renovate edits, weighed against the
  four-eyes bypass it creates.** Owner-less patterns after the catch-all for
  every ecosystem Renovate manages (both `.yml` and `.yaml`); evaluated from
  the base branch; validate the file literally (`codeowners/errors` is not
  sufficient). Approval bots run from default-branch code (`workflow_run`)
  and their workflow file stays owned. A recorded bot approval is an audit
  artefact; a ruleset bypass actor is the absence of a control. (medium, repo
  settings)
- **Scope automerge with `matchUpdateTypes` on the rule, not with
  `:automergeMinor` inside it.** The preset scopes only `automerge`; every
  other key in that rule (`labels`, `assignees`, `autoApprove`) still applies
  to majors. A rule's `automerge: true` never beats an update-type block:
  `major: { automerge: false }` from a preset flattens after rules and wins.
  (high) Verify: `simulate` a major of a matched package.
- **Automerge does what its `description` says.** A rule with
  `automerge: true`, a broad matcher and neither `matchUpdateTypes` nor
  `matchJsonata` is a finding unless the description says majors are intended
  — deliberate `matchJsonata: ["isBreaking = true"]` automerge exists. The
  check is "matches the stated intent", not "never automerges majors".
  `simulate` cannot evaluate `matchJsonata` (it reports `no-match` with
  `readFields: []` for every update type on 44.42.1), so `automerge: false`
  from such a run is a false negative: use `get_provenance automerge` plus the
  rule text, or a real `renovate --dry-run`, and label the finding a
  hypothesis. (high)
- **No `automerge: false` rules that restate the default**, no
  `platformAutomerge: false` beside `automerge: false`, no alias presets whose
  body is a single `extends`. (low)
- **Groups multiply blast radius; two green PRs can break `main` together.**
  A weekly automerged group reds the default branch for every later group.
  Per-PR CI tests each change against the base at branch time, not the
  combination (`typescript@7` and `@typescript-eslint/eslint-plugin@8.67`
  each merged green; together they violate the plugin's peer range). After
  that the artifact step fails on every branch, Renovate commits
  `package.json` alone and the lockfile rots. Grouping plus automerge needs a
  required PR check that exercises every member, or a smaller group, or
  review. (high)
- **Before automerging into an IaC repo**, make the apply pipeline safe for
  batched, unattended pushes: per-root `concurrency`, an
  `event.before..event.after` diff range, a lock timeout, a scheduled
  reconcile. (medium)
- **A change of `extends` governs every open Renovate PR on the next run.**
  Before switching a repo onto an overlay that automerges more, list the open
  `renovate/*` PRs and hold or accept the ones that were deliberately open;
  put the behavioural deltas (automerge scope, dropped groups, limits,
  schedule) in the PR body. (medium)
- **Keep bot PRs out of paid or rate-limited per-PR automation** (hosted AI
  review, quota'd checks): filter on author type and trigger once, not per
  push. (low)

## Grouping and noise

- **Group by upstream, not by name.**
  `matchSourceUrls: ["https://github.com/backstage/backstage"]` catches every
  package published from the monorepo; a templated `groupName`
  (`Backstage plugin {{ lookup (split packageName '-') 1 }}`) keeps plugin
  families together. (low)
- **Do not fight the built-in groups blindly.** `config:recommended` already
  groups ~700 monorepos; a later `groupName` on the same packages silently
  takes over. No separate `@types` group: types travel with their library.
  Check with `simulate` which rule wins. (medium)
- **Compose `group:*` presets at a preset's top level**, never inside a rule
  (that is the warning). A preset that flips an upstream rule
  (`pinDigests: false` against `helpers:pinGitHubActionDigests`) must appear
  **after** `config:best-practices` in `extends`: rules concatenate in order
  and later rules win per key. (medium)
- **Versions that must move together across managers share one `groupName`**
  whose matcher reaches both spellings (mise `cargo:` tools and `Cargo.toml`
  crates; a CLI slug in `mise.toml` and Maven coordinates in
  `smithy-build.json`), with a CI equality check behind it. Otherwise one copy
  drifts and the equality check turns the next automerged bump red. (medium)
  Verify: `simulate_group` one update per manager → one group.
- **Rate limits match the review capacity.** `prHourlyLimit: 2` is Renovate's
  own default (`get_provenance` → `defaults`, not a preset). The maintainer's
  preset lifts it (`prHourlyLimit: 0`) and bounds concurrency
  (`prConcurrentLimit: 20`). Low limits plus a dashboard hide updates in
  "Pending Approval"; unlimited plus no automerge floods. (low)
- **`schedule:*` presets over raw cron, `timezone` set, per-subset schedules
  inside the rule.** `schedule` is not mergeable: a second top-level schedule
  preset **replaces** the first repo-wide instead of adding a lane; a class of
  updates with its own cadence gets `schedule` inside its `packageRules`
  entry. Without `timezone` everything is UTC. With automerge and limits a
  schedule is usually unnecessary, and an unscheduled `lockFileMaintenance` is
  already weekly. (low) Verify: `get_provenance schedule` shows override, not
  concat.
- **Dependency dashboard on** (default via `config:recommended`): pending,
  rate-limited and errored updates become visible there. It is not an
  inventory — any dependency with a `skipReason` (bare-SHA action refs
  included) is filtered out of "Detected dependencies" silently; audit the
  files or `--dry-run=extract` for those. (low)
- **PR-title and commit linters that see Renovate PRs disable header length**
  (`header-max-length: [0]`, `body-max-line-length: [0]`) and keep the type
  and scope rules: grouped titles enumerate every dependency and exceed 100
  and 120 characters; do not exempt the bot instead. Set
  `semanticCommits: "enabled"` (or `:semanticCommits`) where such a lint
  gates merges — the default `auto` relies on history detection and flips in
  mixed repos. (low)

## Rules hygiene

- **Every rule has a `description` saying what the rule is for.** Where the
  justification and link live is an org style choice: a repo-local rule can
  carry the issue link; a shared preset repo may cap descriptions (purpose
  only, ~150 characters, CI-enforced) and keep the reasoning in the README
  and PR body, because descriptions surface on the dashboard and in the PR
  bodies of every consumer. `comment` is not a key. (low)
- **Every rule has at least one matcher of its own.** A matcher-less rule
  applies to everything. The validator error is `Each packageRule must contain
  at least one match* or exclude* selector`, but for a rule that extends a
  relative preset it degrades to the warning `this rule extends a relative
  preset that cannot be resolved during validation, so its selectors could not
  be checked` — keep a selector directly on such rules. A selectors-only rule
  is the warning `must contain at least one non-match* or non-exclude* field`
  (a no-op). (high)
- **Matchers read fields the dependency has.** A rule whose only matcher is
  `matchSourceUrls` or `matchCategories` on a datasource that supplies no
  `sourceUrl`, or on a manager with the wrong category, never fires.
  `matchFileNames` also tests every `lockFiles` entry, so a negation fails
  when a shared lock file matches. Verify with `simulate` and read
  `missingInputs`. (medium)
- **Allowlist first; plain strings over regexes that cannot fire; exemptions
  as narrow as the manager's fields allow.** An explicit `matchPackageNames`
  / `matchSourceUrls` membership derived from real data beats `!`-negation
  sweeps or `ignoreDeps`. For github-actions `["Org/repo"]` and
  `["/^Org/repo(/|$)/"]` are identical (`packageName` is `owner/repo`): write
  the string. Enumerate repositories instead of `Org/**`; use
  `matchDepTypes: ["workflow"]` (>= 44.43.0) when only reusable workflows are
  meant. `matchUpdateTypes: ["!major"]` over the seven-item list (which also
  misses `replacement` and `lockFileMaintenance`); `matchJsonata:
  ["isBreaking = true"]` for "majors and any 0.x change" in one clause.
  (medium) Verify: `compare_simulations` the two spellings → `identical`.
- **Rules are ordered general → specific**, since later rules win, and no
  rule is fully shadowed by a later one setting the same keys on the same
  matches. (medium)
- **`versioning` in a rule only to override the datasource default**; equal
  to the default it is dead config. (low)
- **Custom managers are proven by extraction, use `managerFilePatterns`,
  marker comments in the `customManagers:*` convention, and
  `matchManagers: ["custom.regex"]`.** Prefer the built-in manager — confirmed
  to exist in the consumer's **pinned** Renovate, not latest — then the
  `customManagers:*` presets with a `# renovate:` marker on the line above the
  `_VERSION` value, then a regex manager copied from the upstream marker
  regex. Every hard-coded tool version in CI without a marker is a finding.
  `extract_deps` finds which built-in managers claim a file; it cannot run a
  `customType: regex` manager — prove those by running the `matchStrings`
  against the real file or with `renovate --dry-run=extract`
  (`../../edit-config/reference/custom-managers.md`). (medium)
- **`postUpgradeTasks` tolerate drift and cover the whole group.** A command
  that fails on an unrelated snapshot turns every bump into a
  `renovate/artifacts` failure (the "Artifact update problem" block moves to a
  PR comment when the body is truncated; the branch is retried only on a
  package-file change, a conflict, the rebase checkbox or a `rebase!` title).
  In a grouped branch only `upgrades[0]`'s `postUpgradeTasks` run (default
  `executionMode: "branch"`; the first upgrade is chosen by
  `fileReplacePosition`, then `depName`), so one rule covering exactly the
  group's package set must carry the superset command, or the group is split.
  Narrow `fileFilters`; self-hosted `allowedCommands` anchored (`^just x$`);
  recipes must run without a TTY. (medium) Verify: `simulate` each group
  member → identical `postUpgradeTasks`.
- **Lockstep pairs Renovate can only half-bump are disabled by file**, naming
  the coupled file and where the manual procedure lives in `description`
  (`matchFileNames: ["rust/dylint/rust-toolchain.toml"], enabled: false`).
  The rule is path-bound: a restructure silently un-disables it unless
  `matchFileNames` and the description move with the file. (medium)
- **`semanticCommitType` is pinned on rules whose bump must cut a release.**
  Renovate's default is `fix(deps)` for `dependencies` and `chore(deps)` for
  devDependencies, so under semantic-release every merged `fix(deps)` bump is
  a patch release. (low)
- **Nothing hand-pushed under `branchPrefix` (`renovate/`).** Two cases: a
  commit by an author not in `gitIgnoredAuthors` on a Renovate branch stops it
  rebasing (push the fix and merge as-is; letting Renovate recreate the branch
  discards it); a foreign branch under the prefix is pruned at the end of
  every run — autoclosed and deleted when unmodified or by an ignored author,
  otherwise retitled `<title> - abandoned` with an "Autoclosing Skipped"
  comment (observed within 7 minutes). Only `renovate/lock-file-maintenance`
  and `renovate/reconfigure` are exempt; `renovate/configure` is the
  onboarding branch and is always claimed. Rollouts use another prefix.
  (medium)
- **Before editing a config or workflow, check who owns the file.**
  `.projen/files.json` listing the renovate file means `.projenrc.ts` is the
  source (change both in one commit; a `.projen/` dir with only `tasks.json`
  means hand edits stick); a first-line banner (`generated`, `do not edit`)
  plus a freshness check means fix the generator; a
  `{ matchManagers: ["github-actions"], matchFileNames: [...], enabled: false }`
  rule makes markers in those files moot. (medium)

## Preset repositories

Detail in `../../edit-config/reference/preset-repos.md`.

- **Root-anchored `/path/name` references between the repo's own files**
  (`./x`, `../x` stay legal but break when the referencing file moves);
  absolute references in the repo's own `renovate.json`, in `description`
  strings meant for consumers, in inherited config and in `globalExtends`.
  Relative refs inherit source, repo and `#tag` (never the parent's
  parameters), so a consumer's `#branch` resolves the whole chain. One that
  cannot be resolved is left raw, warned once (`Could not resolve relative
  preset reference`) and fails **every** consumer with `Relative preset
  reference cannot be resolved (<entry>). ...`; the consumer's escape hatch is
  `ignorePresets: ["<entry exactly as written>"]`. Needs Renovate >= 44.29.0 in
  the runner **and** the CI validator. (high)
- **`ignorePresets` compares the resolved absolute string** — source prefix,
  `//path`, inherited `#tag` — and a consumer's `ignorePresets` fully replaces
  the preset's own, so a catalog never relies on self-ignoring. (medium)
- **`#tag` pins are tracked only for `github>` / `gitlab>` / `gitea>`** by the
  `renovate-config` manager; `local>` and relative refs never, and `npm>` /
  `https://` presets ignore `#tag` altogether. `github>` hardcodes
  `api.github.com`: on GHES use `local>`. (medium) Verify: `extract_deps` with
  `fileName: renovate.json` and the consumer's `extends`.
- **Land the preset PR (and tag, if consumers pin) before any consumer PR.** A
  consumer pointing at a path that exists only on a feature branch fails its
  whole config with `config-validation` and stops Renovate for that repo; a
  `#branch` inside `packageRules[].extends` proves the consumer change
  meanwhile. Bump the runner and the validator pin past a feature's version
  floor before the catalog uses it. (high)
- **Single-purpose presets under thin roots.** One preset finds, groups,
  automerges, schedules or decides trust — never a mix (`automerge: true` plus
  `schedule` in one preset is a smell); roots carry the upstream base plus
  non-optional `system/` presets and no automerge or schedule;
  behaviour-free `packages/` selector presets are extended from inside rules;
  team overlays carry deltas only; generic labels are set once in the roots
  (`addLabels: ["dependencies", "{{{updateType}}}"]`). No single-package holds
  or carve-outs in a shared preset — they belong in the consuming repo,
  published as a copy-paste block. A preset file's path is its public API:
  reword the description, do not rename the file without a shim. (medium)
- **`overrideDescription` on collection presets whose summary must survive**;
  a preset of only `description` + `extends`, or only `description` +
  `matchPackageNames`, loses its description. No `ignoreDeps: []` sentinel
  copied from upstream. (low)
- **`onboardingConfig.extends` is written absolute**: a relative ref there is
  canonicalised **with** the inherited tag into every onboarded repo. (low)
- **CI validates every preset file** (`--strict --no-global`, files
  discovered dynamically), inlines same-repo `packageRules[].extends` from the
  working tree (the validator resolves those against the default branch, so
  an introducing PR cannot otherwise be green), validates bare-selector
  presets wrapped in a one-element `packageRules`, and resolves a consumer
  config at the pushed branch (`run_config` of
  `{"extends": ["github>org/repo#<branch>"]}`) compared against `main` on a
  representative dependency. `accepted: true` means nothing while
  `stageStatus.preset` is `error` or `presetErrors` is non-empty. (high)

## Anti-patterns seen in the wild (verified against Renovate 44.x unless marked)

| pattern                                                                                             | what actually happens                                                                                                                   | fix                                                                                          |
| --------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------- |
| `matchPackageNames: ["*", "!gradle"]`                                                               | validation error on current Renovate                                                                                                    | `["!gradle"]`                                                                                |
| `extends: ["group:jackson"]` inside a rule                                                          | warning; nested `packageRules`; works via migration side effect; duplicate                                                              | `extends: ["monorepo:jackson"]` + `groupName` + `matchUpdateTypes`                           |
| `extends: [":automergeMinor"]` + `labels` in one rule                                               | labels on majors too                                                                                                                    | `matchUpdateTypes: ["minor","patch"]` on the rule                                            |
| `automerge: true` on a rule under a preset's `major: { automerge: false }`                          | the update-type block flattens after rules and wins; the rule is inert for majors                                                       | scope with `matchUpdateTypes`, or change the block                                           |
| `automerge: false` rule restating the default; `platformAutomerge: false` beside `automerge: false` | no-op                                                                                                                                   | delete                                                                                       |
| `matchUpdateTypes: ["lockFileMaintenance"]` rule                                                    | overrides `:maintainLockFilesWeekly` instead of composing                                                                               | top-level `lockFileMaintenance: {…}`                                                         |
| `matchUpdateTypes` + `rangeStrategy` in one rule                                                    | refused: `packageRules cannot combine both matchUpdateTypes and rangeStrategy`                                                          | two rules                                                                                    |
| `matchUpdateTypes: ["bogusvalue"]`, `labels: [1, true]`                                             | validator passes; only `$schema` catches it; the clause never matches                                                                   | keep `$schema`; check spellings with `simulate`                                              |
| `matchCategories: ["python"]` to group python tools                                                 | matches every `pre-commit` hook (manager category)                                                                                      | `matchManagers`                                                                              |
| `ignorePaths` in a repo to skip `test/` for .NET                                                    | nuget's manager block replaces the array; `test/` stays managed                                                                         | rule with `matchFileNames: ["test/**"]`, `enabled: false`                                    |
| `comment: "…"` / `ignoreFileName`                                                                   | config refused / option does not exist                                                                                                  | `description` / `matchFileNames`                                                             |
| `matchManagers: ["regex"]`                                                                          | migrated silently to `custom.regex` on every run (same class as `fileMatch`)                                                            | write `custom.regex`                                                                         |
| `fileMatch`, `regexManagers`, `matchPackagePatterns`, `stabilityDays`, `masterIssue`, `config:base`, `managerBranchPrefix` | migrated silently every run; `transitiveRemediation` is deleted (option removed, fails `--strict`)                       | write the migrated form                                                                      |
| `extractVersion` to drop a `-alpha` suffix for changelogs                                           | writes a version that does not exist into the manifest                                                                                  | `versionCompatibility`                                                                       |
| `rangeStrategy: "pin"` under a `regex:` versioning                                                  | silent no-op, reset to `replace`                                                                                                        | drop it                                                                                      |
| `groupName: ["NodeJS"]`                                                                             | migrated to a string, with a warning                                                                                                    | `"NodeJS"`                                                                                   |
| `addLabels: ["{{{upgradeType}}}"]`                                                                  | unknown template field renders empty, no warning; every PR gets an empty label                                                          | `{{{updateType}}}`                                                                           |
| version duplicated in `Dockerfile ARG` and `mise.toml` (or `mise.toml` and `smithy-build.json`)     | the copy drifts; a CI equality check turns the next automerged bump red                                                                 | one source managed by its manager; if two are unavoidable, both extracted and one `groupName` |
| `@scope/**` glob for a monorepo group                                                               | sweeps in `@scope/darwin-arm64` platform binaries                                                                                       | explicit names or `matchSourceUrls`                                                          |
| `minimumGroupSize` at the root                                                                      | gates every group                                                                                                                       | put it on the rule                                                                           |
| `force` in a repo config                                                                            | warning says ignored; merge applies it                                                                                                  | remove it                                                                                    |
| top-level `abandonmentThreshold` under `config:best-practices`                                      | `abandonments:recommended`'s `matchPackageNames: ["*"]` rule sets `1 year` over it                                                      | a trailing `matchPackageNames: ["*"]` rule, last in the file                                 |
| `uses: o/r@<sha>` with no comment, or a prose comment                                               | `skipReason: unversioned-reference`: unmanaged, absent from the dashboard, no rule rescues it                                            | `# vX.Y.Z` (or `# main` to follow the branch)                                                |
| `# v7` comment on a SHA pin                                                                         | `currentValue: v7`; hides which release the digest is; movement inside the major compared at major granularity                          | the exact release of the same SHA (`# v7.1.1`)                                               |
| URL in `matchPackageNames` under github-actions (`!https://github.com/actions{/,}**`)               | packages are `owner/repo`; the negation excludes nothing; the rule fires for official actions too                                        | `actions/**`, or `matchSourceUrls`                                                           |
| `matchPackageNames: ["Org/*/.github/workflows/*"]`                                                  | the sub-path is not part of the name; matches nothing                                                                                   | `Org/repo` (or `matchDepTypes: ["workflow"]` on >= 44.43.0)                                  |
| `pinDigests: false` expecting existing pins to revert                                               | stops future pins only; Renovate re-pins within hours if the rule lands after the revert                                                | merge the rule first, revert by hand, close re-proposed pin PRs                              |
| runtime "download latest, fail if older than N days" script                                         | gates on the publisher's cadence; fails every build once no release is young enough                                                     | pinned `_VERSION` + `# renovate:` marker + `minimumReleaseAge` + an automerge rule           |
| `npx <tool>` (or `pnpm dlx`) in a CI step                                                           | resolved at run time: outside the lockfile and both release-age gates                                                                    | exact pin in `package.json` + `npm ci` / `pnpm exec`                                         |
| duplicate or nested npm `overrides` for one package                                                 | `EOVERRIDE` in Renovate's artifact step; `package.json` committed without the lockfile; `npm ci` on `main` fails                         | one top-level rule per package, audited for reachability                                     |
| `renovate/stability-days` or `renovate/artifacts` as a required check                               | exist only on Renovate branches (the second only on failed ones); every human PR is blocked                                              | drop; require the CI aggregator                                                              |
| `allowedVersions` hold for a major a required check already fails                                   | hides the upgrade; someone has to remember to lift it                                                                                   | delete the hold; let the PR stay red                                                         |
| a hold recorded only in a README or code comment                                                    | Renovate opens the major on the next run                                                                                                | `allowedVersions` rule with `description` — when CI cannot catch the break                   |
| `extends` entry already reached transitively                                                        | the preset is resolved twice (`treeSummary.duplicates`)                                                                                 | delete the entry                                                                             |
| the same preset under two spellings (`org/repo`, `local>org/repo`, `github>org/repo`, `//default`, `:base` vs `//base`) | exact-string dedupe: `packageRules`, `addLabels` and every mergeable array concatenated twice                       | one spelling per preset                                                                      |
| `ignorePresets: ["./x"]`, or an untagged string against a tagged chain                              | never matches; the preset is merged silently                                                                                            | the resolved string with source prefix, `//path` and `#tag`                                  |
| `npm>pkg#1.2.3`, `https://…/default.json#v1`                                                        | `#tag` ignored (npm: `dist-tags.latest`, the tag folds into the name and fails; http: the whole string is the URL)                      | `github>`/`gitlab>` for anything that must be pinnable                                       |
| `local>org/repo#v2.0.0` expecting bump PRs                                                          | the `renovate-config` manager tracks tags only on `github>`/`gitlab>`/`gitea>` (`unsupported-datasource`)                              | `github>org/repo#v2.0.0`                                                                     |
| `.github/dependabot.yml` kept next to Renovate                                                      | duplicate update PRs                                                                                                                    | remove it once `vulnerabilityAlerts` is confirmed on                                         |
| commitlint `header-max-length` on PR titles / squash commits                                        | grouped Renovate titles exceed 100 and 120 characters; every group PR fails the lint                                                     | `header-max-length: [0]`, `body-max-line-length: [0]`; keep type/scope rules                 |
| a hand-pushed `renovate/*` branch                                                                   | pruned at the end of the run: autoclosed, or retitled `- abandoned` with an "Autoclosing Skipped" comment                                | another branch prefix                                                                        |
