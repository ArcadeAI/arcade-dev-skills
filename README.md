# Arcade AI Skills

Ship your internal APIs, services, and data sources as production-grade MCP tools on [Arcade](https://arcade.dev) -- with AI doing the heavy lifting.

## What is this?

Arcade provides 100+ pre-built integrations (Gmail, Slack, GitHub, Salesforce, etc.), but your organization has its own APIs and services that agents need to reach. These skills teach your AI coding assistant how to build custom MCP tools that plug into Arcade's runtime -- giving your internal services the same secure auth, hosting, and governance as the built-in ones.

Drop a skill into Cursor, Claude Code, or any AI coding agent, and it handles the scaffolding, auth patterns, and deployment for you.

## Available skills

| Skill | What it helps you do |
|-------|---------------------|
| [custom-mcp](custom-mcp/) | Turn your internal APIs and services into MCP tools deployable on Arcade -- handles project setup, OAuth/API key auth, agentic design patterns, and cloud deployment via `arcade deploy` |

## Setup

### Cursor

Copy the skill folder into your Cursor skills directory:

```bash
# Personal (available in all your projects)
cp -r custom-mcp ~/.cursor/skills/

# Or project-level (shared via git)
mkdir -p .cursor/skills
cp -r custom-mcp .cursor/skills/
```

The skill activates automatically when you ask Cursor to build an MCP tool or anything matching its trigger terms. You can also reference it explicitly:

> *"Build a custom MCP tool for this Swagger spec and deploy it to Arcade"*

### Claude Code

Add the skill as a project instruction:

```bash
# Copy into your project
cp -r custom-mcp /path/to/your/project/

# Then reference it in your prompt
```

> *"Follow the instructions in custom-mcp/SKILL.md to wrap our internal API as an MCP tool with API key auth"*

### Other agents

Any AI agent that can read files can use these skills. Point it at `SKILL.md` and tell it to follow the instructions.

## Skill structure

Each skill is a directory with a main `SKILL.md` and optional reference files:

```
custom-mcp/
├── SKILL.md               # Main instructions (loaded first)
├── patterns-reference.md   # 54 agentic tool design patterns
└── templates.md            # Complete starter templates (OAuth, API key, inline)
```

The agent reads `SKILL.md` first, then pulls in the reference files on demand.

## Links

- [Arcade Documentation](https://docs.arcade.dev)
- [Arcade Tool Patterns](https://arcade.dev/patterns)
- [Arcade MCP Framework](https://github.com/ArcadeAI/arcade-mcp)
