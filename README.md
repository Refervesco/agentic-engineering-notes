# Agentic Engineering Notes

Field notes on building real software with AI coding agents, with the measurements to back them up.

These are not opinion pieces.
Each one comes from a production codebase (the admin panel and data layer of a small two-repo e-commerce operation), built solo with Claude Code, and where there is a claim there is a number behind it.

## Start here: Putting CLAUDE.md on a Context Diet

An AI coding agent loads one instruction file into its context on every single turn.
Mine had quietly grown to 269 lines and roughly 85 KB: a few real rules buried under a duplicated schema reference and an iteration-by-iteration changelog.

I slimmed it and ran a clean A/B (same code, same model, two branches, docs the only variable).

| | Before | After |
|---|---|---|
| The instruction file | ~85 KB / 269 lines | 15.5 KB / 155 lines |
| Always-loaded memory | 37.8k tokens | 9.2k tokens (**-76%**) |
| Startup context | 59.9k tokens | 31.4k tokens (**-48%**) |
| Answer quality (blind A/B) | baseline | no degradation |

That is roughly 28k tokens saved on every turn, for the life of every session, with no loss in answer quality.

Read it: [context-diet.md](./context-diet.md)

## The method behind it: HAS

The diet is one move inside a larger way of working I call HAS (a Holacratic Agentic System): how I run many independent agent sessions against one codebase without an orchestrator.

Two ideas do the heavy lifting:

- **Governance by written rules, not a boss agent.** Authority lives in an explicit, version-controlled constitution every session is bound by, with the human as final arbiter.
- **Coordination by traces, not messages (stigmergy).** Parallel sessions never talk to each other; they coordinate through marks left in a shared substrate (git, the project tracker, a memory layer), and stale marks decay so the signal stays honest.

The principle the context diet applies is **single owner per concern**: every fact lives in exactly one place, chosen by how it is consumed.
Rules you must obey every turn stay in the always-loaded file; deep reference, data shapes, history, and status each move to the one place that owns them.

## About

Built and written by Refervesco (solo developer, full-stack plus the AI-agent tooling around it).

I am currently open for work opportunities in AI-native and agent-assisted engineering.
If your team is figuring out how to build seriously with coding agents, get in touch:

- LinkedIn: https://www.linkedin.com/in/maxime-vonthron/
- Email: maxime@refervesco.com

## License

Text is shared under CC BY 4.0. Use it, quote it, just credit it.
