# /dig — What Your AI Forgot

You have months of buried decisions sitting on your hard drive right now.

Every Claude Code session you've ever run is stored locally as a transcript. Inside those transcripts are the decisions you made along the way — why you picked this library over that one, the constraint you adopted in February, the approach you tried, abandoned, and (three weeks later) almost tried again. Claude's context was compacted and cleared dozens of times since; the reasoning is gone from its memory. But it's still verbatim on your disk.

`/dig` is a free Claude Code skill that mines that history and recovers it:

```
Dug through 47 sessions (Jan 12 – Jul 5).
Your AI's context was compacted 19 times along the way.
Recovered 43 decisions → DECISIONS.md
31 of them are written down nowhere else.
Most re-litigated: auth token storage (decided 3 times).
```

You walk away with a `DECISIONS.md` for your actual project — every recovered decision, dated, with the reasoning *as stated at the time*, superseded decisions kept and marked (the graveyard stops you re-litigating them). Plus the report above, if you want to share how much your AI forgot.

## Install

```bash
mkdir -p ~/.claude/skills/dig && curl -fsSL https://raw.githubusercontent.com/getscribld/scribld-dig/main/SKILL.md -o ~/.claude/skills/dig/SKILL.md
```

Requires a Claude Code version with Skills support (v2+). If a slash command isn't recognized, update Claude Code first.

Or copy [SKILL.md](SKILL.md) into `~/.claude/skills/dig/SKILL.md` by hand. Then, inside any project you've worked on:

```
/dig
```

Scope it if you like: `/dig the last 10 sessions`, `/dig everything`, `/dig since March`.

## What counts as a decision

The skill is opinionated — that's the point. It looks for six things: choices between alternatives, durable constraints, reversals (with what triggered them), conscious deferrals, contract/naming choices, and — the highest-value category — moments where *you overrode the AI*. Your corrections encode preferences that exist nowhere in your code. It explicitly skips routine implementation steps, narration, and anything readable from the code itself. Every recovered entry must trace to a verbatim quote in your transcripts; if it can't be quoted, it isn't reported.

## Honest notes

- **Privacy:** everything runs locally inside your own Claude Code. Your transcripts are never uploaded, and this skill makes no network calls.
- **Cost:** a deep dig is a real run — it's your Claude Code doing the reading. Expect a few minutes and meaningful token use on a long history. Scope it down if you want a taste first.
- **Format drift:** session transcripts are an undocumented format. The skill fails soft (skips what it can't parse and tells you), and it will say so plainly if the format has shifted rather than hand you an empty report.
- **Thin history:** fewer than a handful of sessions and there's nothing to dig. Start with the hand-maintained templates in [dev-docs-starter](https://github.com/getscribld/dev-docs-starter) instead.
- **Your history is evaporating:** Claude Code prunes session transcripts after ~30 days by default (`cleanupPeriodDays`). Whatever's older than that is already gone — decisions you want recovered should be dug before the timer gets them.

## Companion tools

`/dig` recovers the backlog; these keep it from building up again — all free, all local:

- **[scribld-checkpoint](https://github.com/getscribld/scribld-checkpoint)** — `/checkpoint` before you `/compact`: goal, state, decisions, dead ends, next step, written to a local log.
- **[scribld-context-passport](https://github.com/getscribld/scribld-context-passport)** — generates a portable `CONTEXT.md` that cold-starts ChatGPT/Cursor/Gemini on your project.
- **[dev-docs-starter](https://github.com/getscribld/dev-docs-starter)** — the hand-maintained templates, including the same `DECISIONS.md` format `/dig` writes into.

## Why this exists

Built by [Scribld](https://scribld.io) — persistent, searchable project memory for AI power users. `/dig` recovers what your AI already forgot; Scribld is the live version — it keeps your decision log current automatically, searchable from every AI tool you use (Claude, ChatGPT, Cursor, Gemini) via one shared memory. The skill is free, standalone, and useful without an account. If the report card's "written down nowhere else" number bothers you, you know where to find us.

MIT licensed.
