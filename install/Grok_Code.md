# BYTEX skill for Grok Code (grok-build-0.1)

The [`bytex-trading-engine`](../bytex-trading-engine/) folder at the root of this repository is an Agent Skill: `SKILL.md` plus reference files. It lets Grok Code work with the open-source [BYTEX](https://github.com/BYTEX-TRADE/bytex) trading engine from a chat: write a strategy, backtest it, load market data and paper-trade it.

## Install

- **Grok Build (the `grok` CLI):** copy the `bytex-trading-engine` folder to `~/.grok/skills/` (all projects) or `.grok/skills/` in a repository. Grok also reads Claude Code skills, so an existing `.claude/skills/` install works too. Run it with `/bytex-trading-engine` or let Grok pick it by its description.
- The model formerly called `grok-code-fast-1` was retired on 15 May 2026; requests to it are routed to `grok-build-0.1`.

The skill clones the engine and runs its CLI, so the machine needs `git` and the .NET 10 SDK.

## Official documentation this follows

- https://docs.x.ai/build/features/skills-plugins-marketplaces
- https://docs.x.ai/build/features/project-rules
- https://docs.x.ai/developers/models/grok-build-0.1
