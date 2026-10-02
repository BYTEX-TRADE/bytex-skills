# BYTEX skill for GLM (Z.ai)

The [`bytex-trading-engine`](../bytex-trading-engine/) folder at the root of this repository is an Agent Skill: `SKILL.md` plus reference files. It lets GLM work with the open-source [BYTEX](https://github.com/BYTEX-TRADE/bytex) trading engine from a chat: write a strategy, backtest it, load market data and paper-trade it.

## Install

- **Claude Code on GLM (GLM Coding Plan):** configure Claude Code with the Z.ai Anthropic-compatible endpoint as described in the Z.ai guide, then copy the `bytex-trading-engine` folder to `~/.claude/skills/`.
- **ZCode:** copy the folder to `~/.zcode/skills/`, or import it from your Claude Code skills in Settings > Skills. Type `$` in the chat input and pick `bytex-trading-engine`, or let it trigger from the description.

The skill clones the engine and runs its CLI, so the machine needs `git` and the .NET 10 SDK.

## Official documentation this follows

- https://docs.z.ai/devpack/tool/claude
- https://docs.z.ai/devpack/tool/others
- https://zcode.z.ai/en/docs/skill
