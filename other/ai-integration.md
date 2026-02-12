# AI Integration

Ecotone provides multiple ways for AI assistants and code editors to access up-to-date documentation. This ensures AI tools give accurate, current answers about Ecotone Framework.

## Ecotone Skills

Ecotone provides **Agent Skills** that give AI coding agents deep knowledge of Ecotone's patterns, conventions, and best practices. Skills follow the open [Agent Skills standard](https://agentskills.io/specification) (SKILL.md), making them compatible with a growing number of coding agents.

When installed, Ecotone Skills are automatically activated based on context — the AI agent detects when you're working with Ecotone and loads relevant guidance for Message Handlers, Aggregates, Event Sourcing, Asynchronous Processing, Workflows, Testing, and more.

**Skills repository:** [github.com/ecotoneframework/skills](https://github.com/ecotoneframework/skills)

### Claude Code

Install Ecotone Skills as a plugin:

```
/plugin install ecotone-skills@ecotone
```

This installs skills, slash commands, and subagents tailored for Ecotone development. Manage the plugin with:

```
/plugin disable ecotone-skills@ecotone
/plugin enable ecotone-skills@ecotone
/plugin uninstall ecotone-skills@ecotone
```

### Cursor

Cursor supports the SKILL.md standard natively. Install using the Vercel skills CLI:

```bash
npx skills add ecotoneframework/skills -a cursor
```

Or place Ecotone skills manually in your project:

```
.cursor/skills/<skill-name>/SKILL.md
```

### Windsurf

Windsurf's Cascade agent supports Agent Skills. Place skills in:

```
.windsurf/skills/<skill-name>/SKILL.md
```

### GitHub Copilot

GitHub Copilot supports Agent Skills in VS Code. Place skills in:

```
.github/skills/<skill-name>/SKILL.md
```

### Cline

Cline supports skills since version 3.48.0. Enable them in **Settings → Features → Enable Skills**. Skills are placed in the standard `.claude/skills/` directory.

### Augment Code

Augment Code discovers skills automatically from multiple directories:

```
.augment/skills/<skill-name>/SKILL.md
.claude/skills/<skill-name>/SKILL.md
```

### Kiro

Kiro supports skills at user and workspace level:

```
.kiro/skills/<skill-name>/SKILL.md
```

### OpenAI Codex CLI

Codex CLI reads skills from `~/.codex/skills/` and must be started with the `--enable skills` flag.

## MCP Server (Model Context Protocol)

The MCP server provides direct access to Ecotone documentation for AI assistants that support the Model Context Protocol.

**MCP Server URL:**

```
https://docs.ecotone.tech/~gitbook/mcp
```

### VSCode / Cursor Integration

Click the link below to instantly add Ecotone MCP to your VSCode or Cursor:

[Install Ecotone MCP in VSCode](vscode:mcp/install?%7B%22name%22%3A%22Ecotone%22%2C%22url%22%3A%22https%3A%2F%2Fdocs.ecotone.tech%2F~gitbook%2Fmcp%22%7D)

Or manually add to your MCP settings:

```json
{
  "servers": {
    "Ecotone": {
      "url": "https://docs.ecotone.tech/~gitbook/mcp"
    }
  }
}
```

### Claude Desktop

Add to your Claude Desktop MCP configuration (`claude_desktop_config.json`):

```json
{
  "mcpServers": {
    "Ecotone": {
      "url": "https://docs.ecotone.tech/~gitbook/mcp"
    }
  }
}
```

## Context7 Integration

[Context7](https://context7.com) provides up-to-date documentation for LLMs and AI code editors. Ecotone documentation is available through Context7.

### Using Context7 MCP

Add Context7 MCP to access Ecotone and other library documentation:

```bash
npx -y @upstash/context7-mcp
```

Then query Ecotone documentation through your AI assistant with Context7 enabled.

## LLMs.txt

Ecotone follows the [llmstxt.org](https://llmstxt.org) specification for AI-friendly documentation.

**Available files:**

| File                                                | Description                               |
| --------------------------------------------------- | ----------------------------------------- |
| [llms.txt](https://ecotone.tech/llms.txt)           | Concise overview with key links           |
| [llms-full.txt](https://ecotone.tech/llms-full.txt) | Extended documentation with code examples |

These files are automatically discovered by AI crawlers via `robots.txt`.

## Best Practices for AI-Assisted Development

When using AI assistants with Ecotone:

1. **Enable MCP** - Connect the Ecotone MCP server for real-time documentation access
2. **Reference specific features** - Ask about specific patterns like "Ecotone Saga" or "Ecotone Event Sourcing"
3. **Include context** - Mention your framework (Symfony/Laravel) for framework-specific guidance
4. **Verify code** - Always test AI-generated code against the official documentation

## Supported AI Tools

Ecotone documentation integrates with:

| Tool | MCP | Skills | llms.txt |
| ---- | --- | ------ | -------- |
| **Claude Code** | Yes | Yes (`/plugin install`) | Yes |
| **Cursor** | Yes | Yes (`.cursor/skills/`) | Yes |
| **Windsurf** | Yes | Yes (`.windsurf/skills/`) | Yes |
| **GitHub Copilot** | Yes | Yes (`.github/skills/`) | Yes |
| **Cline** | Yes | Yes (Settings → Enable Skills) | — |
| **Augment Code** | — | Yes (`.augment/skills/`) | — |
| **Kiro** | — | Yes (`.kiro/skills/`) | — |
| **OpenAI Codex CLI** | — | Yes (`~/.codex/skills/`) | — |
| **ChatGPT** | — | — | Yes |
| **Context7-enabled tools** | Yes | — | — |

## Resources

* [Ecotone Documentation](https://docs.ecotone.tech)
* [Ecotone Blog](https://blog.ecotone.tech)
* [GitHub Repository](https://github.com/ecotoneframework/ecotone-dev)
* [Discord Community](https://discord.gg/GwM2BSuXeg)

## Materials

## Links

* [Vibe coding Enterprise PHP Applications](https://blog.ecotone.tech/vibe-coding-enterprise-php-applications/)
