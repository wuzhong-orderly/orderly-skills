# Orderly Network SDK Skills

A comprehensive collection of focused "skill" guides for AI agents building DEX applications with the Orderly Network SDK. Each subfolder contains a `SKILL.md` describing an actionable task an agent can execute.

---

## Skills Overview

### Getting Started

| Skill | Description |
|-------|-------------|
| [create-dex-from-template](create-dex-from-template/SKILL.md) | Clone and configure the Orderly One no-code template |
| [orderly-sdk-install-dependency](orderly-sdk-install-dependency/SKILL.md) | Install SDK packages and dependencies |
| [orderly-sdk-dex-architecture](orderly-sdk-dex-architecture/SKILL.md) | Project structure, providers, and configuration |

### Core SDK

| Skill | Description |
|-------|-------------|
| [orderly-sdk-hooks](orderly-sdk-hooks/SKILL.md) | React hooks for account, trading, positions, orders |
| [orderly-sdk-ui-components](orderly-sdk-ui-components/SKILL.md) | UI component library (buttons, inputs, tables, etc.) |
| [orderly-sdk-page-components](orderly-sdk-page-components/SKILL.md) | Pre-built pages (Trading, Portfolio, Markets) |

### Integration

| Skill | Description |
|-------|-------------|
| [orderly-sdk-wallet-connection](orderly-sdk-wallet-connection/SKILL.md) | Wallet integration (EVM + Solana), authentication |
| [orderly-sdk-trading-workflows](orderly-sdk-trading-workflows/SKILL.md) | Complete trading flows (deposit → trade → withdraw) |

### Customization & Debugging

| Skill | Description |
|-------|-------------|
| [orderly-sdk-theming](orderly-sdk-theming/SKILL.md) | Custom colors, fonts, logos, branding |
| [orderly-sdk-debugging](orderly-sdk-debugging/SKILL.md) | Error handling, troubleshooting, debugging |

### Advanced

| Skill | Description |
|-------|-------------|
| [low-level-api](low-level-api/SKILL.md) | Direct REST/WebSocket API without SDK |

---

## Quick Start

### For AI Agents

Point your agent at this workspace and reference the relevant `SKILL.md` files based on the task:

```
"Build a DEX" → create-dex-from-template + orderly-sdk-dex-architecture
"Add trading" → orderly-sdk-hooks + orderly-sdk-trading-workflows
"Fix styling" → orderly-sdk-theming
"Debug error" → orderly-sdk-debugging
```

### For Developers

1. Start with [orderly-sdk-install-dependency](orderly-sdk-install-dependency/SKILL.md) for packages
2. Follow [orderly-sdk-dex-architecture](orderly-sdk-dex-architecture/SKILL.md) for setup
3. Use [orderly-sdk-page-components](orderly-sdk-page-components/SKILL.md) for quick pages
4. Customize with [orderly-sdk-hooks](orderly-sdk-hooks/SKILL.md) for custom features

---

## Using with Claude Agent Skills

1. Read the Agent Skills docs: https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview
2. Use `/claude/skill-template.yaml` as a starting point
3. Reference specific `SKILL.md` files in your instructions

## Using with VS Code Custom Agents

1. Read the VS Code docs: https://code.visualstudio.com/docs/copilot/customization/agent-skills
2. Use `/.vscode/agent-skill-sample.json` as a template
3. Point the agent at this workspace

---

## Resources

- **Sample DEX**: See `Resources/sample-dex/` for a complete reference implementation
- **SDK Source**: See `Resources/js-sdk/` for the SDK source code
- **Documentation**: See `Resources/documentation/` for additional docs

## Sample Agent Configs

- VS Code: [.vscode/agent-skill-sample.json](.vscode/agent-skill-sample.json)
- Claude: [claude/skill-template.yaml](claude/skill-template.yaml)