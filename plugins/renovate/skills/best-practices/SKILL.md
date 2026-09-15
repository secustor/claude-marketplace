---
name: best-practices
description: Review a Renovate configuration against proven practices and report what to change — preset choice (config:recommended vs config:best-practices), supply-chain hardening (digest pinning with exact-version comments, minimumReleaseAge floors and first-party exemptions, the vulnerability-alert gap, no npx in CI), automerge safety (what `automerge: true` actually arms; required checks, rulesets and bypass actors, merge queues, CODEOWNERS, push-only workflows), GitHub Actions pinning and reusable workflows, grouping, rate limits, lock file maintenance, holds versus red PRs, deprecated syntax, preset-repo hygiene — with every finding verified through the renovate-config-debugger tools instead of from memory, and findings the tools cannot compute labelled as hypotheses. Use when asked to review, audit, harden, clean up or set up a renovate.json / renovate.json5 / preset repo, when a Renovate PR merged something it should not have or did not automerge, or for "is this config good".
---

# Renovate best practices review

A review is only worth reading if each finding is true for *this* config on
*this* Renovate version. Resolve first, then judge. `run_config` with the
`rcd` MCP tools (from the `renovate-config-debugger` plugin this plugin
depends on) returns the verdict, the digest and a `runId`; `get_provenance`
tells you whether a value is set by the repo, a preset or a default;
`get_preset_node` shows what a preset actually contains on the pinned version;
`get_option_docs` gives the option's meaning and default for that version.
No MCP: `npx -y @renovate-config-debugger/cli digest|validate|provenance|docs`.

## Procedure

1. **Resolve.** `run_config` the file (and the org preset it extends, if the
   user maintains that too). Read `stageStatus.preset` and `presetErrors`
   **before** `accepted`: `accepted: true` with a preset-stage error means the
   config was validated with that preset expanded into nothing, and every
   provenance or simulate answer describes an empty layer. Supply
   `RCD_GITHUB_TOKEN`, or read the preset with
   `gh api "repos/O/R/contents/<file>.json?ref=<tag>" --jq .content | base64 -d`,
   and otherwise label findings on that layer as hypotheses. Then note the
   warnings, the digest's "rewrote N deprecated options", and
   `treeSummary.duplicates`: a duplicate is a finding — an `extends` entry the
   chain already reaches, or one preset under two spellings, doubling every
   mergeable array — unless it is structural
   (`helpers:pinGitHubActionDigestsToSemver` next to `config:best-practices`
   resolves `helpers:pinGitHubActionDigests` twice; say so instead of flagging
   it).
2. **Read the effective config, not the file.** `get_provenance` without a key
   lists every option some layer set and who won. Compare that list against
   `reference/checklist.md`. A practice the org preset already supplies is not
   a finding for the repo; a repo option the preset also sets is.
3. **Check redundancy honestly.** An option equal to its default is redundant
   only if nothing earlier in the chain sets the opposite; `:automergeDisabled`,
   `:enableRenovate`, `:renovatePrefix` and about a dozen more built-ins are
   pure reset-to-default presets. Say "no-op unless something upstream set
   X", not "redundant". Update-type blocks flatten **after** rules: a preset's
   `major: { automerge: false }` beats any rule's `automerge: true`, and
   `get_provenance` cannot see inside blocks — `simulate` can.
4. **Simulate the risky paths.** Anything that automerges: read
   `reference/automerge-gates.md` first, then `simulate` the update types the
   rule's `description` claims and excludes. The check is "does what the
   description says", not "never automerges majors" — deliberate
   `matchJsonata: ["isBreaking = true"]` automerge exists. `simulate` cannot
   evaluate `matchJsonata` (it reports `no-match` with `readFields: []` for
   every update type), so `automerge: false` from such a run is a false
   negative: fall back to `get_provenance automerge` plus the rule text, or a
   real `renovate --dry-run`, and label the finding a hypothesis. Anything
   that disables: `simulate` a dependency that must stay enabled. Anything
   that groups: `simulate_group` the members. A `pinDigests: false`
   exemption: `simulate` with `updateType: "pinDigest"`. A release-age
   exemption: `simulate` with and without `sourceUrl` and read
   `missingInputs`.
5. **Separate config from repo settings.** Required checks, rulesets and
   their bypass actors, merge queues, CODEOWNERS, `allow_auto_merge`,
   workflow triggers and the package manager's own cooldown decide whether
   automerge merges and whether a red PR is even possible. None of that is in the config: report it under "Not
   verified" with the `gh` recipe from `reference/automerge-gates.md`, or run
   the recipe when the user has `gh` access to the repo.
6. **Report** with `reference/report-format.md`. Severity first, one verdict
   sentence per finding, the evidence line (rule index as both repo and merged
   index, preset name that wrote the value, Renovate version), then the exact
   edit. State version-gated facts (the `workflow` depType from 44.43.0,
   relative preset refs from 44.29.0, merge-queue handling without platform
   auto-merge from 44.73.0) against the **consumer's** pinned version — read
   from the runner's `package.json` — not the debugger's.

Do not present a finding that rests on a matcher whose input you did not
supply, on a hedge about what "may" happen, or on option semantics you
recalled instead of looked up. Do not colour a verdict you have not computed.

## What "good" looks like

The maintainer's shared preset (`local>secustor/renovate-config`) is the
worked example behind the checklist. Re-resolve it (`run_config` +
`get_preset_node`) rather than quoting this list as its content. What it
demonstrates:

- a baseline of `config:recommended`, with `config:best-practices` as the
  stated direction for repos that can take digest-pinning PRs;
- GitHub Action digests pinned with the semver comment kept exact
  (`helpers:pinGitHubActionDigestsToSemver`), and `# renovate:` markers
  honoured in Dockerfiles and workflows (`customManagers:*` presets);
- a release-age gate for third-party npm, with the org's own packages
  exempted by `matchSourceUrls` rather than swept in;
- automerge granted only by scoped rules that carry a `description` — dev
  dependencies where a test suite exists, GitHub Actions, package managers
  and Node on `matchUpdateTypes: ["patch", "minor"]` — never a broad grant;
- PR limits set to the review capacity (`prConcurrentLimit`,
  `prHourlyLimit: 0`) instead of Renovate's default hourly cap that parks
  updates in "Pending Approval";
- per-repo files that only `extends` the preset, plus
  `lockFileMaintenance: { enabled: true, automerge: true }` and
  `postUpdateOptions: ["pnpmDedupe"]` where the lockfile warrants it.

Read `reference/checklist.md` for the practices with their reasons and the
anti-patterns, `reference/automerge-gates.md` before judging anything that
automerges, `reference/github-actions.md` for pins, comments, reusable
workflows and `uses-with` inputs, and `reference/presets.md` for what the
recommended presets contain on the pinned version.
