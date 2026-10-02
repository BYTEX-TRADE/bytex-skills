# BYTEX skill for Gemini Flash

The [`bytex-trading-engine`](../bytex-trading-engine/) folder at the root of this repository is an Agent Skill: `SKILL.md` plus reference files. It lets Gemini Flash work with the open-source [BYTEX](https://github.com/BYTEX-TRADE/bytex) trading engine from a chat: write a strategy, backtest it, load market data and paper-trade it.

## Install

- **Gemini CLI:** copy the `bytex-trading-engine` folder to `~/.gemini/skills/` (all projects) or `.gemini/skills/` in a workspace (`.agents/skills/` works as an alias). Or link it in place: `gemini skills link ./bytex-trading-engine`. Check with `/skills list`. Gemini activates it through `activate_skill` and asks you to approve.
- **Gemini app (Gems):** create a Gem and add `SKILL.md` and the `references/` files under Knowledge. A Gem cannot run the CLI, so it gives you commands to run instead.

The skill clones the engine and runs its CLI, so the machine needs `git` and the .NET 10 SDK.

## Official documentation this follows

- https://geminicli.com/docs/cli/skills/
- https://geminicli.com/docs/cli/creating-skills/
- https://support.google.com/gemini/answer/15146780
