# Formahand Skills

[Agent Skills](https://agentskills.io) that teach coding agents (Claude Code, Codex, Cursor and any agent that reads skills) how to work with a [Formahand](https://formahand.com) store: setting it up, designing the storefront, products and orders, shipping, payments readiness, apps, store functions, integrations, events and webhooks, environments and the money rules that stay with the merchant.

This folder is generated from the Formahand platform (its tool registry and its public docs) and is the mirror published as the public `formahand/skills` repository. Do not edit it by hand: change the source and rebuild.

## Skills

| Skill | Description |
| --- | --- |
| [formahand](./formahand) | Work with a Formahand store through its MCP tools and API. Use when working with a Formahand store: set up, design the storefront, products, shipping, payments readiness, apps, store functions, integrations, events, webhooks, environments, money. Covers the first calls of every session, the draft to publish loop, the walls and money decisions that stay with the merchant, and which reference to read for each task. |
| [reference-storefront-theme](./reference-storefront-theme) | Use a live website, screenshot, or URL as inspiration to design and build a merchant-editable Formahand storefront theme. Covers reference discovery, capability checks, safe asset use, Formahand sections and behaviors, and evidence-based verification. |

The same content is served at https://formahand.com/skills/formahand/SKILL.md, with its references under `https://formahand.com/skills/formahand/references/` and an index at https://formahand.com/skills.
The reference-theme workflow is included in this repository for agents to install when they need it.

## Installation

Every client gets the same skill; only the way it is installed differs.

### Any agent: the skills CLI

```bash
npx skills add formahand/skills --skill formahand
```

For reference-inspired storefront work, also install the companion skill:

```bash
npx skills add formahand/skills --skill reference-storefront-theme
```

Add `-g` to install it for every project rather than the current one. When the CLI asks which agents to install for, pick yours (Claude Code, Codex, Cursor and others are supported).

### Claude Code: plugin marketplace

```
/plugin marketplace add formahand/skills
/plugin install formahand
```

### Codex

Codex loads skills from `.agents/skills` in the repository you work in (and in each folder between your working directory and the repository root) and from `~/.agents/skills` for every repository. Any of these works:

1. Run the skills CLI above in the project and choose Codex. It writes the skill to `.agents/skills/formahand`.
2. Copy it by hand:

   ```bash
   git clone https://github.com/formahand/skills.git
   mkdir -p ~/.agents/skills
   cp -R skills/formahand ~/.agents/skills/formahand
   ```

3. Ask Codex's built-in installer: `$skill-installer install the formahand skill from https://github.com/formahand/skills`.

Restart Codex if the skill does not appear. To make sure Codex reads it before touching a store, add this to the project's `AGENTS.md`:

```markdown
## Formahand

Before any work on the Formahand store, read the `formahand` skill
(`.agents/skills/formahand/SKILL.md`, or https://formahand.com/skills/formahand/SKILL.md if it is not
installed) and follow it: make its first five calls, and never add a payment
method, choose a paid package or pay for anything.
```

### By hand, for any other agent

Copy `formahand/` into the skills folder your agent reads (for example `~/.claude/skills/`, `~/.cursor/skills/` or `~/.agents/skills/`), or point it at https://formahand.com/skills/formahand/SKILL.md.

## Connecting the agent to a store

The skill tells the agent how to work; a store API token lets it act. Mint one in the dashboard under Settings, Agents & API, then:

```bash
export FORMAHAND_KEY=<your token>
claude mcp add formahand --transport http https://formahand.com/mcp --header "Authorization: Bearer $FORMAHAND_KEY"
```

Everything else, including Codex and generic MCP configuration, is in the skill and at https://formahand.com/docs/agents.
