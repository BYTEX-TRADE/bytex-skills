# BYTEX skill for Qwen

The [`bytex-trading-engine`](../bytex-trading-engine/) folder at the root of this repository is an Agent Skill: `SKILL.md` plus reference files. It lets Qwen work with the open-source [BYTEX](https://github.com/BYTEX-TRADE/bytex) trading engine from a chat: write a strategy, backtest it, load market data and paper-trade it.

## Install

- **Qwen Code:** copy the `bytex-trading-engine` folder to `~/.qwen/skills/` (all projects) or `.qwen/skills/` in a project. Qwen invokes it by its description, or run `/bytex-trading-engine`; `/skills` lists what is loaded.
- **Qwen Chat:** has no documented skills upload. Paste `SKILL.md` and the reference file you need into the conversation; it will give you commands to run instead of running them.

The skill clones the engine and runs its CLI, so the machine needs `git` and the .NET 10 SDK.

## Official documentation this follows

- https://qwenlm.github.io/qwen-code-docs/en/users/features/skills/
