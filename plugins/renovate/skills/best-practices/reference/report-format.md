# How to write the review

Persona studies on the debugger found the same three things carried readers
of every skill level to a correct conclusion: one plain verdict sentence
first, the per-clause evidence next, and every number framed by where it came
from. The same rules apply to prose a reviewer pastes into a PR.

## Shape

```
## Renovate config review — <file> (resolved with Renovate <version>)

Accepted: yes / yes with N warnings / NO (Renovate would refuse this config)
Resolved: <n> presets, <r> rules (<k> from this file, <r-k> from `<extends entry>`)

### Must fix
1. **<verdict sentence>** — `<option or packageRules[i] (merged [j])>`, set by <layer>.
   <one sentence of mechanism>. Fix:
   ```json5
   …
   ```
   Verified: <simulate/compare result in one line, with the dependency used>.

### Should fix
…

### Consider
…

### Not verified
- <finding> — needs <what: a sourceUrl, the org preset, repo settings>.
```

## Rules

- **Verdict first, evidence second.** "Major updates of `express` would
  automerge" beats "there may be an issue with the automerge rule".
- **Flat statements about this config.** Not "this is safe when every
  remaining entry is a negation" when, in this config, every remaining entry
  *is* a negation. Say "safe: the remaining entries are all negations".
- **Name the writer, not just the index.** `packageRules[0] of
  security:minimumReleaseAgeNpm`, not `packageRules[726] of
  config:best-practices`. Give the repo index and the merged index when both
  exist; Renovate's own messages use the merged, 0-based one.
- **Default versus set.** `automerge: false (Renovate default, not set in the
  file)` is a different claim from `automerge: false (set by packageRules[3])`.
  Do not attribute a default to the config.
- **Frame counts.** "734 rules: 1 from this file, 733 from
  `local>secustor/renovate-config`". Unframed counts read as "did I break
  something".
- **Say what the simulation had.** A `no-match` decided by a clause whose
  field you left unset is not evidence; report it under "Not verified" with
  the field it needs.
- **Warnings are not errors, and errors are not style.** Three states:
  accepted clean, accepted with warnings, refused. A refused config gets one
  finding at the top and nothing below it until it is fixed.
- **No hedging on option semantics.** Look them up (`get_option_docs`); if the
  option does not exist on the pinned version, say that, and name the option
  that does.
- **Do not repeat by-design behaviour as a finding.** A preset losing its
  wrapper description, `config:recommended` enabling the dashboard, monorepo
  groups: mention them only where they answer the user's question.
- **A finding you did not run a check for is a hypothesis.** Label it.
