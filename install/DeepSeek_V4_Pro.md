# BYTEX skill for DeepSeek V4 Pro (deepseek-v4-pro)

The [`bytex-trading-engine`](../bytex-trading-engine/) folder at the root of this repository is an Agent Skill: `SKILL.md` plus reference files. It lets DeepSeek V4 Pro work with the open-source [BYTEX](https://github.com/BYTEX-TRADE/bytex) trading engine from a chat: write a strategy, backtest it, load market data and paper-trade it.

## Install

- **Claude Code on DeepSeek:** configure Claude Code with DeepSeek's Anthropic-compatible endpoint as described in the DeepSeek guide, choosing `deepseek-v4-pro` as the model, then copy the `bytex-trading-engine` folder to `~/.claude/skills/`.
- **DeepSeek Harness (developer preview):** copy the folder to `~/.agents/skills/` (all projects) or `.agents/skills/` in a repository.

The skill clones the engine and runs its CLI, so the machine needs `git` and the .NET 10 SDK.

## Official documentation this follows

- https://api-docs.deepseek.com/quick_start/agent_integrations/claude_code
- https://api-docs.deepseek.com/quick_start/pricing
- https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/subsystems/skills.md
