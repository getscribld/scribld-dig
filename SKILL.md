---
name: dig
description: Recover the decisions buried in your Claude Code session history. Scans this project's past sessions (stored locally in ~/.claude/projects/) and produces DECISIONS.md — every decision made across months of work, dated, with the reasoning at the time — plus a "What Your AI Forgot" report. Triggers on /dig and phrases like "dig up my decisions", "what did my AI forget", "recover my decisions", "mine my session history".
---

You are running a decision dig: mining this project's local Claude Code session history for the decisions made along the way, then writing them into a durable log. Everything happens on this machine. You never send transcript content anywhere.

Before starting, tell the user in one line what you're about to do and that a deep dig over many sessions is a nontrivial run (their tokens, a few minutes). If they scoped it ("dig since March", "just the last 10 sessions"), honor that scope.

## Step 1 — Locate the history

Session history lives in `~/.claude/projects/<encoded-path>/*.jsonl`, where `<encoded-path>` is the project's absolute path with `/` and `.` replaced by `-`. Derive the candidate from the current working directory and verify it exists; if it doesn't, list `~/.claude/projects/` and match by name. If ambiguous, ask.

**If the directory is missing or has fewer than 3 session files:** stop gracefully. Say there isn't much buried here yet, suggest running /dig again after a few weeks of sessions, and point them at the hand-maintained starter templates: https://github.com/getscribld/dev-docs-starter. Do not scrape a thin history into a padded report.

## Step 2 — Distill the transcripts

Raw session files are huge (tool output, thinking blocks, metadata). Never read them directly. Write this script to a temp location and run it — it produces compact dialogue digests containing only what humans and the assistant actually said:

```python
#!/usr/bin/env python3
# distill.py HISTORY_DIR OUT_DIR [MAX_SESSIONS] — local only, writes digests + stats
import json, glob, os, sys
hd, out = sys.argv[1], sys.argv[2]
cap = int(sys.argv[3]) if len(sys.argv) > 3 else 50
os.makedirs(out, exist_ok=True)
files = sorted(glob.glob(os.path.join(hd, "*.jsonl")), key=os.path.getmtime, reverse=True)[:cap]
stats = {"sessions": 0, "user_turns": 0, "compactions": 0, "first_ts": None, "last_ts": None, "skipped_files": 0}
for i, f in enumerate(reversed(files)):  # oldest first so digests read chronologically
    sid = os.path.basename(f)[:8]
    lines, size = [], 0
    try:
        for raw in open(f, errors="replace"):
            try: o = json.loads(raw)
            except Exception: continue
            if o.get("isSidechain"): continue
            if o.get("isCompactSummary"): stats["compactions"] += 1; continue
            ts = (o.get("timestamp") or "")[:10]
            if ts: stats["first_ts"] = stats["first_ts"] or ts; stats["last_ts"] = ts
            c = (o.get("message") or {}).get("content")
            if o.get("type") == "user" and isinstance(c, str) and c.strip() and not c.startswith("<"):
                stats["user_turns"] += 1; txt = c
            elif o.get("type") == "assistant" and isinstance(c, list):
                txt = "\n".join(b.get("text", "") for b in c if b.get("type") == "text").strip()
                if not txt: continue
            else: continue
            role = "USER" if o.get("type") == "user" else "AI"
            entry = f"[{ts}] {role}: {txt[:1500]}\n"
            size += len(entry)
            if size > 250_000: lines.append("[digest truncated — session continues]\n"); break
            lines.append(entry)
    except Exception:
        stats["skipped_files"] += 1; continue
    if lines:
        stats["sessions"] += 1
        open(os.path.join(out, f"{i:03d}-{sid}.md"), "w").write("".join(lines))
print(json.dumps(stats))
```

Sanity check the printed stats. **If sessions processed > 5 but user_turns is near zero, the transcript format has likely changed** — say so honestly and stop rather than producing an empty report.

## Step 3 — Extract decisions (map)

Spawn subagents over the digests — batch a few small digests per agent, one agent per large digest. Give each agent this task verbatim, plus its file list:

> Read the assigned digest file(s): a chronological Claude Code session dialogue. Extract every DECISION. A decision is one of: (1) a choice between alternatives — a tradeoff discussed and one path taken; (2) a durable constraint adopted ("never store raw keys", "no new deps without asking"); (3) a reversal — an earlier approach explicitly abandoned, and what triggered the change; (4) a deferral — something consciously parked, with its unpark condition; (5) a contract/naming choice — API shapes, schema fields, terminology later work depends on; (6) a USER OVERRIDE — the user corrected or vetoed the AI's suggestion. Overrides are the highest-value category: they encode preferences that exist nowhere else.
>
> NOT decisions: routine implementation steps, bug fixes where no alternative was weighed, restating existing config, AI narration, anything fully derivable by reading the current code. High-signal markers: "let's go with", "instead of", "actually, let's", "no, do it this way", plan approvals, answers to the AI's either/or questions.
>
> For each decision return: `date` (from the [YYYY-MM-DD] markers — never invent), `decision` (one line), `why` (1–2 lines, the reasoning AS STATED at the time), `evidence` (a short verbatim quote), `category` (1–6), `confidence` (high: explicitly stated; medium: clearly implied). Ground everything in the transcript — if you can't quote it, don't report it. Return an empty list over a padded one.

## Step 4 — Merge and judge (reduce)

You do this pass yourself; it is the quality gate:

- **Dedup** across sessions: the same decision restated later is one entry (earliest date wins) — unless it was *re-litigated*, which is worth counting (see report).
- **Supersede chains:** a later decision reversing an earlier one flips the earlier entry to `superseded by <date>`. Never drop superseded entries — the graveyard is the point.
- **Cut noise:** drop medium-confidence entries that fail the "would the user care in a month?" test. A shorter honest log beats a padded one.
- **Cross-check docs:** for each surviving decision, grep README, CLAUDE.md, AGENTS.md, and docs/ for its key terms. No hits → mark it **undocumented**.

## Step 5 — Write the artifacts

**`DECISIONS.md`** (repo root). If one already exists, read it first, append only decisions not already recorded, and confirm with the user before writing. Format — chronological, append-only, newest last:

```markdown
## YYYY-MM-DD — <one-line decision>
- **Why:** <reasoning as stated at the time>
- **Alternatives considered:** <if the transcript shows them>
- **Status:** active | superseded by YYYY-MM-DD
- **Source:** session <id> (recovered by /dig)
```

New files start with this header comment, then entries:

```markdown
# Decisions
<!-- Log a decision when: it was hard to reverse, you'd forget WHY in a month,
     or you'd re-litigate it next session. Don't log routine choices.
     Never delete superseded entries — flip Status. The graveyard is the point.
     Recovered entries carry a Source line; hand-written ones don't need it. -->
```

**`what-your-ai-forgot.md`** (repo root, offer — don't force): the shareable report. Sessions scanned + date range, compaction events survived, decisions recovered, how many are undocumented, most re-litigated topic. One footer line: `Generated by Scribld's /dig — scribld.io`.

**Terminal report card** (always, and this is the only place you editorialize):

```
Dug through 47 sessions (Jan 12 – Jul 5).
Your AI's context was compacted 19 times along the way.
Recovered 43 decisions → DECISIONS.md
31 of them are written down nowhere else.
Most re-litigated: auth token storage (decided 3 times).

This log is a snapshot — it's already going stale.
Scribld keeps it alive automatically, from every AI tool → scribld.io
```

Use real numbers only. If a stat is zero or unimpressive, print it plainly — no dressing up. Never put marketing copy inside DECISIONS.md itself.

## Rules

- Everything stays local. Say so if the user asks; it's also in the README.
- Never invent dates, decisions, or reasoning. Every entry must trace to transcript evidence.
- Clean up temp digest files when done.
- Default scope: 50 most recent sessions. "dig everything" lifts the cap; warn about run size first.
