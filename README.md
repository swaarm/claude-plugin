# Swaarm plugin for Claude Code

Teach Claude Code how to operate [Swaarm](https://swaarm.com) — the performance marketing and mobile measurement platform — through the Swaarm MCP server at `mcp.swaarm.com`.

With this plugin installed, Claude understands Swaarm's domain model (offers, publishers, advertisers, leadflows, postbacks, conversions, macros, network adapters, optimization and automation rules), speaks Swaarm's vocabulary, and runs the workflows a human operator would run — so it behaves like a knowledgeable teammate, not a generic assistant poking at tool schemas.

## Install

In Claude Code, run:

```
/plugin marketplace add swaarm/claude-plugin
/plugin install swaarm@claude-plugin
```

That's it. The skill auto-activates whenever you mention Swaarm, any Swaarm entity, or invoke a Swaarm MCP tool.

## Updating

```
/plugin marketplace update claude-plugin
```

Or enable auto-update for the `claude-plugin` marketplace and Claude Code pulls new versions on startup.

## What's inside

- `skills/swaarm/` — domain knowledge: concepts, terminology, the MCP tool surface with gotchas, common operator workflows, and persona lenses (account manager, etc.)

## Prerequisites

You need a Swaarm account and access to the Swaarm MCP server at `mcp.swaarm.com`. See [docs.swaarm.com](https://docs.swaarm.com) for MCP connection setup.

## Support

- Documentation: https://docs.swaarm.com
- Issues: https://github.com/swaarm/claude-plugin/issues

## License

TBD