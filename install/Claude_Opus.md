# BYTEX skill for Claude Opus

The [`bytex-trading-engine`](../bytex-trading-engine/) folder at the root of this repository is an Agent Skill: `SKILL.md` plus reference files. It lets Claude Opus work with the open-source [BYTEX](https://github.com/BYTEX-TRADE/bytex) trading engine from a chat: write a strategy, backtest it, load market data and paper-trade it.

## Install

- **Claude Code:** copy the `bytex-trading-engine` folder to `~/.claude/skills/` (all projects) or `.claude/skills/` in a project. Claude loads it when a request matches the description, or run `/bytex-trading-engine`.
- **Claude apps (claude.ai, Desktop):** zip the `bytex-trading-engine` folder so the folder is the root of the zip, then Customize > Skills > + > Create skill > Upload a skill. Code execution must be on, and the skill needs network access to clone the engine.
- **Claude API:** skills run in a container without network access, so the engine cannot be cloned there. Use Claude Code for the full workflow.

The skill clones the engine and runs its CLI, so the machine needs `git` and the .NET 10 SDK.

## Official documentation this follows

- https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview
- https://platform.claude.com/docs/en/agents-and-tools/agent-skills/best-practices
- https://code.claude.com/docs/en/skills
- https://support.claude.com/en/articles/12512198-creating-custom-skills
