# 🏛 council

An [agent skill](https://agentskills.io) that stops your coding agent from winging it: before giving you a final plan, whichever agent you're talking to must **convene the other agent CLIs installed on your machine and deliberate**.

Say:

> council, 2 turns: should we migrate this app from Express to Hono?

and the lead agent (Claude Code, Codex, Cursor, Gemini, Grok, or OpenCode — whoever you asked) will:

1. Draft a plan.
2. Present it **independently** to every other agent CLI on your machine — each member gives exactly **one** reply per turn (verdict: `SHIP IT` / `SHIP WITH CHANGES` / `RETHINK`, strongest objections first). Members never see each other's replies.
3. Revise the plan, telling members what changed — and repeat for as many turns as you asked.
4. Answer with the whole deliberation as a conversation, plus the final plan.

Everything is one-shot CLI invocations under the hood (no ACP, no daemons, no API keys beyond the CLIs you already have installed and logged in). All member invocations are read-only: they may read your repo, never touch it. Failed or missing members are skipped gracefully and noted in the transcript.

## Install

The skill is a single `SKILL.md`. Clone it where your agent looks for skills:

```bash
# Claude Code (personal)
git clone https://github.com/kitze/council ~/.claude/skills/council

# Claude Code (per-project)
git clone https://github.com/kitze/council .claude/skills/council
```

For other agents that support the [Agent Skills](https://agentskills.io) format, clone into their skills directory (e.g. `~/.codex/skills/council`), or just hand your agent the `SKILL.md` as instructions — it's self-contained.

## Requirements

At least one other agent CLI installed and logged in: `claude`, `codex`, `cursor-agent`, `gemini`, `grok`, or `opencode`. The more you have, the bigger the council.

## Notes

- Deliberation sessions (plans, per-round prompts and replies) are kept in `~/.council/` so you can audit what every member actually said.
- `SKILL.md` embeds the exact headless flags each CLI needs (cursor `--trust`, gemini `--skip-trust`, codex read-only sandbox, and the grok `--system-prompt-override` workaround for its silent headless cancellation) — found the hard way so you don't have to.

## License

MIT
