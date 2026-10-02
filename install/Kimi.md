# BYTEX skill for Kimi

The [`bytex-trading-engine`](../bytex-trading-engine/) folder at the root of this repository is an Agent Skill: `SKILL.md` plus reference files. It lets Kimi work with the open-source [BYTEX](https://github.com/BYTEX-TRADE/bytex) trading engine from a chat: write a strategy, backtest it, load market data and paper-trade it.

## Install

- **Kimi Code CLI:** copy the `bytex-trading-engine` folder to `~/.kimi-code/skills/` or `~/.agents/skills/` (all projects), or `.kimi-code/skills/` / `.agents/skills/` in a project. Kimi invokes it by its description, or run `/skill:bytex-trading-engine`.
- **Claude Code on Kimi models:** configure Claude Code with Moonshot's Anthropic-compatible endpoint as described in the Kimi platform guide, then install the skill as for Claude Code (`~/.claude/skills/`).
- **kimi.com:** add `SKILL.md` and the `references/` files to a Project as files. The chat cannot run the CLI, so it gives you commands to run instead.

The skill clones the engine and runs its CLI, so the machine needs `git` and the .NET 10 SDK.

## Official documentation this follows

- https://moonshotai.github.io/kimi-code/en/customization/skills.html
- https://platform.kimi.ai/docs/guide/claude-code-kimi
- https://www.kimi.com/en/help/features/project
