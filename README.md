# claude-agents

Reusable Claude Code slash command agents for managing software projects.
Each agent is a persona with a defined role, decision authority, and context loading strategy.

## Agents

| Command | Role | Authority |
|---|---|---|
| `/coo` | Chief Operating Officer | Orchestrates team; owns decision log and project velocity |
| `/muse` | Strategic Inspiration & Mentor | Advisory only; challenges assumptions and generates ideas |
| `/architect` | Principal Engineer & Architect | **Blocking authority** over all technical decisions |
| `/deployer` | CI/CD & Infrastructure | Owns path from code to production |
| `/marketing` | Marketing & Social Media Manager | Owns positioning, community, and public properties |

## Decision Authority Chain

```
CEO (user)
  ├── COO          → task sequencing, process, project board
  ├── Muse         → advisory only, no veto
  └── Architect    → technical decisions (blocking authority)
        └── Deployer → infrastructure within approved architecture
        └── Marketing → positioning within approved direction (via COO)
```

## Installation

### Option 1: Copy to your dotfiles

```bash
cp agents/*.md ~/.claude/commands/
```

### Option 2: Clone and symlink (recommended)

```bash
git clone https://github.com/saldistefano/claude-agents ~/code/claude-agents
ln -s ~/code/claude-agents/agents ~/.claude/commands
```

### Option 3: Add to an existing dotfiles repo

Copy the `agents/` directory into your dotfiles structure and symlink as appropriate.

## Usage

Once installed, invoke any agent with its slash command in Claude Code:

```
/coo          → get current project status, blockers, next actions
/muse         → challenge current direction, generate ideas
/architect    → validate a technical decision before implementing
/deployer     → plan a deployment, pipeline, or infrastructure change
/marketing    → draft a post, plan a launch, review positioning
```

## Project Context

These agents are designed to load project context from a `kb/` directory following
the [Meridian knowledge base convention](https://github.com/saldistefano/meridian).

Each agent reads specific `kb/*.md` files on startup. If your project uses a different
structure, edit the "Context I Load on Startup" section of each agent file.

## Philosophy

- **CEO is always the human.** Agents advise, inform, and implement. They never override the user.
- **Architect has blocking authority** — nothing gets built until it's technically approved.
- **Blog diary at end of every session** — `marketing` and `coo` both remind you to write it.
- **Agents are project-agnostic** — edit the context-loading section for your project structure.

## Contributing

These agents were built for the [Meridian project](https://github.com/saldistefano/meridian).
PRs welcome for improvements, new agent types, or project-specific variants.
