# Observe Skills

Agent skills for deploying and managing Observe's observability infrastructure. Works with any AI coding agent that supports the [Agent Skills](https://github.com/vercel-labs/add-skill) standard.

## Install

### Claude Code

```bash
# Add this marketplace
claude plugin marketplace add observeinc/skills

# Install the plugin
claude plugin install observe@observe
```

### Other agents

```bash
npx skills add observeinc/skills
```

To install a specific skill:

```bash
npx skills add observeinc/skills -s alert-investigation
npx skills add observeinc/skills -s observe-cli
npx skills add observeinc/skills -s outlier-detection-analysis
```

## Available Skills

Start here — these are the entry-point skills intended for direct use:

| Skill | Description |
|-------|-------------|
| **alert-investigation** | Investigate alerts using systematic SRE methodology to understand issues, assess impact, test hypotheses, and identify root causes with evidence-based analysis |
| **observe-cli** | Use the Observe CLI to investigate production systems and pull telemetry — search/tail logs, query metrics, explore traces, correlate events, and triage alerts |
| **outlier-detection-analysis** | Identify which field values correlate with bad behavior (slowness, errors, anomalies) using phi-coefficient correlation analysis over OPAL |

The following skills are sub-skills called by entry-point skills, but can also be used standalone:

| Skill | Called by | Description |
|-------|-----------|-------------|
| **generate-opal** | `observe-cli`, `outlier-detection-analysis` | Core OPAL skill — dataset kind selection, column selection rules, core syntax, and skill index. Must be loaded before generating any OPAL |

## Prerequisites

The `observe-cli`, `alert-investigation`, and `outlier-detection-analysis` skills use the [Observe CLI](https://github.com/observeinc/cli) for API operations. During development, run it from the repo with `bun dev -- <command>`.

## Compatibility

These skills work with any agent that supports the SKILL.md format:

- Claude Code
- Cursor
- Codex
- OpenCode
- Windsurf
- Any agent supporting `npx skills add`

## Structure

```
.
├── .claude-plugin/plugin.json   # Claude Code plugin manifest
├── .cursor-plugin/plugin.json   # Cursor plugin manifest
├── .mcp.json                    # MCP server definitions (Claude Code)
└── skills/
    ├── alert-investigation/     # Entry point
    │   └── SKILL.md
    ├── observe-cli/             # Entry point
    │   └── SKILL.md
    ├── outlier-detection-analysis/ # Entry point
    │   ├── SKILL.md
    │   └── references/
    └── generate-opal/           # Sub-skill (OPAL generation)
        ├── SKILL.md
        └── references/
```

## License

Use of these skills is governed by the [Observe Skills License](LICENSE). Skills are licensed as Client Software under your Snowflake agreement for Subscription Services, or the [Observe Terms of Service](https://www.observeinc.com/legal) if your agreement does not include Client Software.
