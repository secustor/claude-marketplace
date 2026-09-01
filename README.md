# claude-marketplace

Claude Code plugins used and developed by [secustor](https://github.com/secustor).
The marketplace name is `secustor`, so installs read `<plugin>@secustor`.

## Usage

```
/plugin marketplace add secustor/claude-marketplace
/plugin install renovate@secustor
```

## Plugins

| plugin                                                                              | what it does                                                                                                                                                     |
| ----------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| [renovate](plugins/renovate)                                                        | Edit, review and migrate Renovate configs and preset repos with proven practices; every edit is resolved and simulated through the debugger before it is handed over. Depends on `renovate-config-debugger`. |
| [renovate-config-debugger](https://github.com/secustor/renovate-config-debugger)     | Debug Renovate configs with Renovate's own code — preset expansion, per-key provenance, packageRules simulation and validation as MCP tools, plus the workflow skill. |

## Layout

```
.claude-plugin/marketplace.json   # marketplace manifest
plugins/renovate/                 # plugins hosted in this repository
  .claude-plugin/plugin.json
  skills/<skill>/SKILL.md
  skills/<skill>/reference/*.md
```

Plugins hosted elsewhere (the debugger) are referenced by repository in the
manifest.
