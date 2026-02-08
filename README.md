# orderly-skills

A small collection of focused "skill" guides for agents and automation. Each subfolder contains a `SKILL.md` describing a short, actionable task an agent can execute (for example: install a dependency, scaffold files, or configure an SDK).

Project structure

- `add-feature-page/` — SKILL.md for adding a feature page
- `configure-sdk/` — SKILL.md for configuring an SDK
- `install-dependency/` — SKILL.md for installing dependencies
- `low-level-api/` — SKILL.md for working with low-level APIs
- `scaffold-dex/` — SKILL.md for scaffolding a DEX

Using these skills with Claude (Agent Skills)

1. Read the Agent Skills docs: https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview
2. Use `/claude/skill-template.yaml` as a starting point for a Claude skill; update `instructions` and `tools` to match your environment.
3. Test the skill with simple inputs (for example: "Install lodash") and review any changes before merging.

Using these skills with VS Code custom agents

1. Read the VS Code docs: https://code.visualstudio.com/docs/copilot/customization/agent-skills
2. Use `/.vscode/agent-skill-sample.json` as a starting point for a VS Code agent configuration.
3. Create the custom agent in VS Code, point it at this workspace, and run it to apply the steps in the chosen `SKILL.md`.

Included sample agent configs

- VS Code custom agent template: [/.vscode/agent-skill-sample.json](.vscode/agent-skill-sample.json)
- Claude agent skill template: [/claude/skill-template.yaml](claude/skill-template.yaml)

Quick usage

- In VS Code: create a custom agent and paste the JSON from [/.vscode/agent-skill-sample.json](.vscode/agent-skill-sample.json) as the agent's instruction set. Open the workspace and ask the agent to run the `entrypoint` steps.
- For Claude: copy `/claude/skill-template.yaml` into your Claude agent definition (or use it as a starting point in the console). Update instructions and tools, then test with a small task.