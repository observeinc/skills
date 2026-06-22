# Observe Skills

Agent skills that let any AI coding agent investigate and manage your [Observe](https://www.observeinc.com) data.

Observe is an observability platform for logs, metrics, and traces. These skills teach your AI coding agent how to work with it — investigating incidents, querying telemetry, onboarding data, building visualizations, and more.

## Install

### Claude Code

```bash
# Add this marketplace
claude plugin marketplace add observeinc/skills

# Install the plugin
claude plugin install observe@observe
```

The plugin bundles the Observe MCP server, which needs your **Observe base URL** (e.g. `101.observeinc.com`).

### Cortex Code

```bash
# Run from the repo directory with your Observe base URL
OBSERVE_BASE_URL=<your-base-url> cortex
```

On first run, Cortex will prompt you to complete OAuth in your browser to connect the Observe MCP server. The MCP server and skills are configured automatically from the `.cortex-plugin` directory.

### Codex

Install the skills, then add the Observe MCP server as a separate step:

```bash
# Install the skills
npx skills add observeinc/skills

# Add the Observe MCP server — replace <your-base-url> with your Observe host (e.g. 101.observeinc.com)
codex mcp add observe --url https://<your-base-url>/v1/ai/mcp
```

The server is stored in your global `~/.codex/config.toml`; remove it later with `codex mcp remove observe`.

### Other agents

```bash
# Install all skills without prompts
npx skills add observeinc/skills --all

# Or choose skills interactively
npx skills add observeinc/skills

# Or install a specific skill
npx skills add observeinc/skills -s <skill-name>
```

## Compatibility

These skills work with any agent that supports the [Agent Skills](https://agentskills.io/home) standard, including:

- Claude Code
- Cursor
- Codex
- OpenCode
- Windsurf
- [...and more](https://github.com/vercel-labs/skills#supported-agents)

## License

Use of these skills is governed by the [Observe Skills License](LICENSE). Skills are licensed as Client Software under your Snowflake agreement for Subscription Services, or the [Observe Terms of Service](https://www.observeinc.com/legal) if your agreement does not include Client Software.
