# BYTEX skill for ChatGPT Astra (GPT-6 Astra)

The [`bytex-trading-engine`](../bytex-trading-engine/) folder at the root of this repository is an Agent Skill: `SKILL.md` plus reference files. It lets ChatGPT Astra work with the open-source [BYTEX](https://github.com/BYTEX-TRADE/bytex) trading engine from a chat: write a strategy, backtest it, load market data and paper-trade it.

## Install

- **Codex:** copy the `bytex-trading-engine` folder to `~/.agents/skills/` (all projects) or `.agents/skills/` in a repository. Codex picks it up by its description, or call it with `$bytex-trading-engine`.
- **ChatGPT (Business, Enterprise, Edu):** sidebar Plugins > Skills > Create > Upload from your computer. ChatGPT scans the skill before it becomes available. Call it with `@bytex-trading-engine`.
- **ChatGPT plans without skills:** add `SKILL.md` and the `references/` files to a Project as files. ChatGPT then cannot run the CLI itself and gives you commands to run instead.

The skill clones the engine and runs its CLI, so the machine needs `git` and the .NET 10 SDK.

## Official documentation this follows

- https://developers.openai.com/codex/skills
- https://help.openai.com/en/articles/20001066-skills-in-chatgpt
- https://developers.openai.com/blog/rethinking-skills-and-prompts-for-gpt-6-astra
