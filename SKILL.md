---
name: council
description: Convene a council of AI coding agents before finalizing a plan. The agent you're talking to — whichever CLI is the lead (claude, codex, cursor, gemini, grok, opencode) — drafts a plan, presents it independently to every other agent CLI installed on the machine, each member gives ONE reply per turn, and the lead revises the plan for X turns before answering. Trigger words: council, convene the council, consult the other agents, ask the council, ask codex/cursor/gemini/grok what they think, second opinion from other agents, N turns of deliberation, deliberate before planning.
---

# Council

You are the **lead**. Before giving the user a final plan, deliberate with the other agent CLIs installed on this machine: propose → hear ONE reply from each member → revise, repeated for X turns. Members never see each other's replies — only your plan and, in later rounds, what changed. The deliverable is the final plan plus the deliberation shown as a conversation. Do NOT implement anything unless the user explicitly asked for execution after planning.

## Protocol

**0. Parse.** TASK = the user's ask, verbatim. X = the number of turns they said ("1 turn", "3 turns"...), default 1. X is exact: run all X rounds even if round 1 is unanimous approval, and never add extra rounds. SELF = the product you run in: Claude Code → `claude`, Codex → `codex`, Cursor → `cursor`, Gemini CLI → `gemini`, Grok → `grok`, OpenCode → `opencode`.

**1. Detect members.** Every CLI in the table below whose binary exists (`command -v`) and answers `--version` within ~15s — minus SELF. If the user names members ("council with just codex and grok"), use only those. If none are available, say so and present your solo plan explicitly marked as un-counciled — never fake a council.

**2. Session.** `SESSION=~/.council/$(date +%Y%m%d-%H%M%S)-<short-slug>`, `mkdir -p "$SESSION"`. Write the ask verbatim to `task.md`. Do your normal thinking/exploration, then write your honest first plan to `plan-v1.md` — complete enough to critique: approach, steps, files/tools touched, tradeoffs.

**3. Deliberate — for each round R in 1..X:**

- Build one prompt per member from the template below → `$SESSION/round-R/prompts/<member>.md`.
- Invoke **all members in parallel in one shell command** (background jobs + `wait`), from the project directory the task concerns, each capped at ~300s. Reply → `$SESSION/round-R/replies/<member>.md`, stderr → a log file. macOS has no `timeout` command; use `perl -e 'alarm shift; exec @ARGV' 300 <cmd...>`.
- Read the replies. A member that times out, errors, or returns empty is noted in the transcript ("cursor: no reply — timeout") and the round continues.
- **Revise:** weigh each opinion on its merits — adopt what's right, reject what's wrong (keeping your position is allowed, with reasons). Write `plan-v(R+1).md` (full revised plan) and `changes-v(R+1).md` (what changed and why, what you rejected and why — describe suggestions by content, never by member name: all members see this file next round).

**4. Answer** with the deliberation formatted as a conversation, then the final plan (template at the bottom).

## Member CLIs

`$PF` = that member's prompt file, `$RF` = its reply file. Run from the project directory.

| member   | invocation                                                                                       | why these flags |
|----------|--------------------------------------------------------------------------------------------------|-----------------|
| claude   | `claude -p < "$PF" > "$RF"`                                                                        | print mode auto-denies write/exec tools; reads are fine |
| codex    | `codex exec --skip-git-repo-check -s read-only --color never -o "$RF" - < "$PF" > /dev/null 2>&1`  | read-only sandbox; final message lands in `$RF` via `-o` |
| cursor   | `cursor-agent -p --output-format text --mode plan --trust "$(cat "$PF")" > "$RF"`                  | plan mode is read-only; `--trust` scopes cwd trust to this run (headless refuses untrusted dirs without it) |
| gemini   | `gemini --skip-trust --approval-mode plan -p "$(cat "$PF")" > "$RF"`                               | plan mode is read-only; `--skip-trust` for headless in untrusted dirs |
| grok     | `grok --output-format plain --system-prompt-override "$GROK_SYS" --prompt-file "$PF" > "$RF"`      | see the grok note; binary is sometimes installed as `agent` |
| opencode | `opencode run "$(cat "$PF")" > "$RF"`                                                              | advisory-only enforced at prompt level |

**The grok note.** Headless grok silently cancels the entire run (empty output, exit 0, stream ends `stopReason:"Cancelled"`) the moment the model attempts a tool call — and its stock system prompt makes it attempt one on any work-shaped prompt. Two-part fix, both required: pass `--system-prompt-override` with

> `GROK_SYS="You are one member of a council of AI coding agents, asked for a one-shot written assessment of another agent's plan. Reply directly with plain markdown text following the instructions in the user message. Do not use tools, do not ask questions, do not wait for anything."`

**and** give grok the no-tools closing paragraph in its prompt (below). Grok is the one member that advises without reading the repo.

## Member prompt template

Sections in `[...]` only when they apply. Everything is plain text you substitute.

```
You are <member>, one of several AI coding agents serving on a council.
The lead agent, <self>, received a task from its user and must consult this council before presenting the final plan. This is consultation round <R> of <X>.
Council rules: members reply independently. You do not see other members' replies — only the lead's plan and, in later rounds, how it changed.

## The task (verbatim from the user)

<contents of task.md>

## <self>'s current plan (v<R>)

<contents of plan-vR.md>

[## What the lead changed since the previous round, and why

<contents of changes-vR.md>]

[## Your own reply in the previous round

<contents of round-(R-1)/replies/<member>.md>]

## How to reply

Give exactly ONE reply — your honest expert assessment of this plan:

1. First line: verdict — SHIP IT, SHIP WITH CHANGES, or RETHINK.
2. The 2-5 most important problems, risks, or gaps. Concrete and specific, strongest first.
3. Where you would do it differently: the alternative and why it wins.
4. If the plan is fundamentally sound, say so and name its single weakest point. Do not invent objections to sound useful.

You are advising, not implementing. Reading files in the current directory is fine; do NOT edit files, run state-changing commands, or write full implementations. Under ~400 words, plain markdown, no preamble.
```

For **grok only**, replace the final paragraph with:

```
You are advising, not implementing. You have NO tool access here — base your reply only on the material above; do not attempt to read files or run commands. Under ~400 words, plain markdown, no preamble.
```

## Rules

- **Independence:** one invocation per member per round; never re-query a member within a round, never forward one member's reply to another.
- **Faithful summaries:** in the transcript, include each member's strongest objection even — especially — when you rejected it.
- **No padding:** if the feedback is thin, say so; don't manufacture disagreement or extra rounds.

## Final answer format

```markdown
# 🏛 Council on: <task one-liner>
*Lead: claude · Members: codex, cursor, grok · 2 turns · session: ~/.council/...*

## Turn 1
**claude (lead):** Proposed v1 — <2-3 line summary of the plan>
**codex:** SHIP WITH CHANGES — <2-3 line summary of the reply>
**cursor:** RETHINK — <...>
**grok:** <...>
**claude (lead):** Revised → v2: adopted <...>; rejected <...> because <...>

## Turn 2
...

## Final plan (v3)
<the full final plan>

*Changed my mind on: <...> · Held my ground on: <...> · Full replies: $SESSION/round-*/replies/*
```
