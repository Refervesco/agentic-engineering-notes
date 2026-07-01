# Agentic Engineering Notes

Field notes on building real software with AI coding agents, with the measurements to back them up.

Not opinion, measurement. This first note comes from a production codebase (the admin panel and data layer of a small two-repo e-commerce operation), built solo with Claude Code. Where there is a claim, there is a number behind it.

## 🔬 Start here: Putting CLAUDE.md on a Context Diet

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

## ⚙️ The system behind it: HAS

The diet is one move inside a larger system I am building and testing in the open: multiple independent coding agents working one codebase with no orchestrator, coordinating through durable shared state and explicit written rules instead of messaging each other. I call it HAS (Holacratic Agentic System).

Two ideas do the heavy lifting:

- **Governance by written rules, not a boss agent.** Authority lives in an explicit, version-controlled constitution every session is bound by, with the human as final arbiter.
- **Coordination by traces, not messages (stigmergy).** Parallel sessions never talk to each other; they coordinate through marks left in a shared substrate (git, the project tracker, a memory layer), and stale marks decay so the signal stays honest.

The principle the context diet applies is **single owner per concern**: every fact lives in exactly one place, chosen by how it is consumed. Rules you must obey every turn stay in the always-loaded file; deep reference, data shapes, history, and status each move to the one place that owns them.

## 👋 About

Written by Maxime Vonthron, a product and technology leader (~20 years) working hands-on with AI coding agents and the systems around them.

Currently open to roles and engagements where teams are building seriously with coding agents.

- LinkedIn: https://www.linkedin.com/in/maxime-vonthron/
- Email: maxime@refervesco.com

## License

Text is shared under CC BY 4.0. Use it, quote it, just credit it.
