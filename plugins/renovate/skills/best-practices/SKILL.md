---
name: best-practices
description: Review a Renovate configuration against proven practices and report what to change — preset choice (config:recommended vs config:best-practices), supply-chain hardening (digest pinning, minimumReleaseAge), automerge safety, grouping, rate limits, lock file maintenance, deprecated syntax, org-preset hygiene — with every finding verified through the renovate-config-debugger tools instead of from memory. Use when asked to review, audit, harden, clean up or set up a renovate.json / renovate.json5 / preset repo, or "is this config good".
---

# Renovate best practices review

A review is only worth reading if each finding is true for *this* config on
*this* Renovate version. Resolve first, then judge. `run_config` with the
`rcd` MCP tools (from the `renovate-config-debugger` plugin this plugin
depends on) returns the accepted/warnings verdict, the digest and a `runId`;
`get_provenance` tells you whether a value is set by the repo, a preset or a
default; `get_preset_node` shows what a preset actually contains;
`get_option_docs` gives the option's meaning and default for the pinned
version. No MCP: `npx -y @renovate-config-debugger/cli digest|validate|provenance|docs`.

## Procedure

1. **Resolve.** `run_config` the file (and the org preset it extends, if the
   user maintains that too). Note `accepted`, warnings, `presetErrors`, the
   digest's "rewrote N deprecated options", and `treeSummary.duplicates`.
2. **Read the effective config, not the file.** `get_provenance` without a key
   lists every option some layer set and who won. Compare that list against
   `reference/checklist.md`. For each candidate finding, check the winner: a
   practice the org preset already supplies is not a finding for the repo.
3. **Check redundancy honestly.** An option equal to its default is redundant
   only if nothing earlier in the chain sets the opposite; `:automergeDisabled`,
   `:enableRenovate`, `:renovatePrefix` and about a dozen more built-ins are
   pure reset-to-default presets. Say "no-op unless something upstream set
   X", not "redundant".
4. **Simulate the risky paths.** Anything that automerges: `simulate` a major
   of a matched package and confirm it does *not* automerge. Anything that
   disables: `simulate` a dependency that must stay enabled. Anything that
   groups: `simulate_group` the members.
5. **Report** with `reference/report-format.md`. Severity first, one verdict
   sentence per finding, the evidence line (rule index as both repo and merged
   index, preset name that wrote the value, Renovate version), then the exact
   edit. Findings you could not verify (missing `sourceUrl`, a preset that
   failed to fetch) are labelled as unverified, not dropped and not asserted.

Do not present a finding that rests on a matcher whose input you did not
supply, on a hedge about what "may" happen, or on option semantics you
recalled instead of looked up. Do not colour a verdict you have not computed.

## What "good" looks like (the user's own baseline)

The maintainer's shared preset (`local>secustor/renovate-config`) is the
canonical example of the practices in the checklist:

```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": [
    "config:recommended",
    "customManagers:dockerfileVersions",
    "customManagers:githubActionsVersions",
    "helpers:pinGitHubActionDigestsToSemver"
  ],
  "packageRules": [
    { "matchDatasources": ["npm"], "matchSourceUrls": ["!https://github.com/secustor/**"], "minimumReleaseAge": "3 days" },
    { "description": "Automerge dev dependencies if there are tests", "matchDepTypes": ["devDependencies"], "matchManagers": ["npm"], "automerge": true },
    { "description": "Automerge GitHub actions", "matchManagers": ["github-actions"], "automerge": true },
    { "description": "Automerge package manager updates", "matchPackageNames": ["yarn", "npm", "pnpm"], "matchUpdateTypes": ["patch", "minor"], "automerge": true },
    { "description": "Automerge safe NodeJS updates", "extends": ["group:nodeJs"], "matchUpdateTypes": ["patch", "minor"], "automerge": true }
  ],
  "prConcurrentLimit": 20,
  "prHourlyLimit": 0
}
```

with `config:best-practices` as the stated direction for repos that can take
digest-pinning PRs, and per-repo files that only `extends` the preset plus
`lockFileMaintenance: { enabled: true, automerge: true }` and
`postUpdateOptions: ["pnpmDedupe"]` where the lockfile warrants it.

Read `reference/checklist.md` for the practices with their reasons and the
anti-patterns, and `reference/presets.md` for what the recommended presets
actually contain on the pinned version.
