# Claude Code Skills

Production-ready custom skills for [Claude Code](https://docs.anthropic.com/en/docs/claude-code) — Anthropic's agentic coding tool.

Skills are reusable prompt modules that extend Claude Code with domain-specific expertise. Drop them into your commands directory and invoke them as slash commands or let them auto-trigger based on context.

## Available Skills

### `/figma-design-md` — Figma to Design System Extraction

Generates a precision `DESIGN.md` from any Figma URL, giving you a pixel-accurate design reference document with Tailwind-native values that an AI agent (or developer) can use to implement the design 1:1.

**What it does:**

1. Pulls design context from Figma via the [Figma MCP server](https://www.figma.com/community/plugin/1469177830956498498/figma-mcp-server)
2. Runs a deep codebase audit — greps every color, spacing class, border radius, font weight, and shadow actually in use
3. Cross-references Figma values with your codebase (prefers your existing Tailwind classes over raw pixels)
4. Outputs a structured `DESIGN.md` with 10 sections: color palette with opacity variants, typography hierarchy, component snippets, spacing scale, responsive breakpoints, and more
5. Saves raw Figma source to `figma-sources/` for traceability

**Key features:**

- **99% precision target** — every value is sourced from Figma or verified via codebase grep. Unverified values are flagged
- **Tailwind-native** — outputs `px-6` not `24px`, `rounded-full` not `rounded-[28px]`, matches your project's actual classes
- **Opacity coverage** — documents every opacity variant (`/5`, `/10`, `/20`...) found in the codebase
- **Copy-paste component snippets** — exact Tailwind class strings for buttons, cards, inputs
- **Multi-node support** — analyze multiple Figma frames and merge into one design system doc
- **Auto-triggers** when you paste a Figma URL and ask to implement/build something

**Requirements:**

- [Figma MCP server](https://www.figma.com/community/plugin/1469177830956498498/figma-mcp-server) configured in Claude Code
- A Tailwind CSS project (the skill outputs Tailwind-native values)

---

### `/kamal` — Kamal Deployment Expert

A comprehensive Kamal 2.x deployment skill that turns Claude Code into a deployment expert for your Rails application. Covers the full lifecycle: server setup, configuration, deployment, CI/CD, Dockerfile optimization, and production debugging.

**What it does:**

1. Auto-discovers your project's Kamal configuration (`config/deploy.yml`, secrets, Dockerfile, hooks)
2. Understands your deployment model (registryless vs remote registry, single vs multi-server)
3. Provides expert guidance for every Kamal operation

**Capabilities:**

| Area | Examples |
|------|----------|
| **Server Setup** | Fresh server preparation, Docker install, deploy user creation, SSH hardening, firewall config |
| **Deployment** | Deploy, redeploy, rollback, lock management, multi-environment (`-d staging`) |
| **Server Access** | Log tailing with grep, Rails console (sandbox by default), database console, container SSH |
| **Dockerfile Optimization** | Layer caching analysis, multi-stage builds, BuildKit cache mounts, `.dockerignore` audit |
| **CI/CD Setup** | GitHub Actions workflows with Docker layer caching (GHA), concurrency control, lock management |
| **Security** | Dedicated deploy keys, non-root deploy user, per-environment secrets, branch protection guidance |
| **Debugging** | Systematic approach: container status, logs, proxy, resources, disk, env vars |

**Safety built in:**

- Destructive commands (`kamal app stop`, `kamal remove`, database ops) require user confirmation
- Rails console opens in sandbox mode by default
- Secrets are never exposed in output
- Read-only debugging by default — only mutates when asked

**Auto-triggers** on: `deploy`, `rollback`, `kamal logs`, `kamal console`, or when your project has a `config/deploy.yml`.

---

## Installation

### 1. Clone

```bash
git clone https://github.com/tvcam/claude-code-skills.git ~/claude-code-skills
```

### 2. Symlink into Claude Code's commands directory

```bash
mkdir -p ~/.claude/commands

# Install the skills you want
ln -s ~/claude-code-skills/figma-design-md ~/.claude/commands/figma-design-md
ln -s ~/claude-code-skills/kamal ~/.claude/commands/kamal
```

### 3. Use

```bash
# Invoke directly
claude
> /figma-design-md https://www.figma.com/design/ABC123/MyFile?node-id=1-284
> /kamal deploy

# Or let them auto-trigger from natural language
> build this figma design: https://figma.com/design/...
> check the production logs for errors
> set up CI/CD for kamal deployment
```

## How Skills Work

Each skill is a directory containing a `SKILL.md` file with:

- **Frontmatter** — metadata: name, version, description, allowed tools, trigger conditions
- **Prompt body** — the full instruction set Claude Code follows when the skill activates

Skills can:
- **Auto-detect** context (e.g., a Figma URL or `deploy.yml` in the project) and activate automatically
- **Use MCP tools** (e.g., Figma MCP for pulling design data)
- **Run multi-step workflows** with codebase analysis, file generation, and self-auditing

## Writing Your Own Skills

Use the existing skills as templates. The key sections in a `SKILL.md`:

```yaml
---
name: my-skill
version: 1.0.0
description: |
  What the skill does.
  - MANDATORY TRIGGERS: slash commands or keywords
  - AUTO-DETECT: conditions for automatic activation
allowed-tools:
  - Bash
  - Read
  - Write
  # ... tools the skill needs
---

# Skill Title

## Preamble
{Confirmation message when skill loads}

## Workflow
{Step-by-step instructions}

## Safety Rules
{Guardrails and constraints}
```

## Updating

```bash
cd ~/claude-code-skills
git pull
```

Symlinks mean updates take effect immediately.

## Contributing

Contributions welcome. When submitting a skill:

1. One directory per skill with a single `SKILL.md`
2. No hardcoded credentials, server IPs, or personal information
3. Include clear trigger conditions and safety rules
4. Test with Claude Code before submitting

## License

MIT
