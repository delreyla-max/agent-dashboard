# Cowork Handoff — New Computer Setup

**As of:** 2026-06-14  
**Written by:** Claude (outgoing session, old computer)  
**For:** Claude (incoming session, new computer)

Read this once at the start of the first session on the new machine. Then proceed normally per CLAUDE.md.

---

## What this system is

Garman uses Cowork (Claude desktop) as an agent manager. The setup lives in `Documents\Claude\` (synced via OneDrive). Claude reads CLAUDE.md at session start, checks the Inbox for tasks, delegates to subagents (Sonnet/Haiku), and delivers results to `3_Results\`. Garman's preferences, tone rules, and durable facts live in flat .md files at the folder root (not in a `_Profile/` subfolder, despite what CLAUDE.md says, that path was never created).

Key files to read every session:
- `CLAUDE.md` — the full playbook
- `about-me.md` — who Garman is
- `how-to-speak.md` — tone rules, banned words, structure rules. Follow these strictly.
- `memory.md` (in `GitHub/`) — durable cross-session facts
- `github-deploy.md` — deploy credentials and rules

---

## The dashboard

A single-file HTML app that tracks agent tasks, job search pipeline runs, and job matches. It's the main artifact of the system.

**Source of truth:** `Documents\Claude\GitHub\agent-dashboard\index.html`  
**Live URL:** https://delreyla-max.github.io/agent-dashboard/ (also gyip.com/agent-dashboard, noted as a privacy risk)

The dashboard embeds all its data in a `<script id="dashboard-data">` JSON block inside the HTML. Claude edits that JSON directly after each task event.

**Publishing rule:** Garman pushes via GitHub Desktop (his review step). If he asks Claude to push: clone to `/tmp`, copy the file, verify it ends with `</html>` and passes `node --check`, then push. Never run git operations inside the mounted `GitHub\` folder. OneDrive sync races corrupt `.git` — verified incident 2026-06-11 (empty .git/config, truncated index.html shipped live).

**GitHub account:** delreyla-max  
**Token:** in `github-deploy.md` (scoped to agent-dashboard and portfolio repos, Contents R/W, expires ~Sep 2026)

---

## Supabase plan (next step, not yet done)

The dashboard currently saves jobs to localStorage, which means saves are per-browser and don't persist across devices or users. The plan is to replace this with Supabase for proper database-backed saves.

A complete setup guide was written on 2026-06-14: `3_Results\2026-06-14_supabase-setup.md`

Summary of what's needed:
1. Create a Supabase project (free tier), run the SQL migration in that file
2. Enable Google OAuth in Supabase, configure redirect URLs
3. Paste `SUPABASE_URL` and `SUPABASE_KEY` (~line 875 in index.html) from Supabase → Project Settings → API
4. Push via GitHub Desktop

The index.html already has placeholder stubs for these values from the session where this was designed. Once the Supabase project is created and the two values pasted in, it's live. No backend code, no server, no API keys exposed.

---

## Connected pipelines

**daily-job-search** — scheduled task, runs at 4:08am daily. Writes `Job Search\Job Searches\jobs_[YYYY-MM-DD].txt`. Claude reads this file at each session start and surfaces the top 3 Tier 1 matches in the daily briefing. Claude does NOT modify this task or its output files.

If the today's jobs file is missing after 5am, flag it as a failed run (LinkedIn rate-limiting is the usual cause).

---

## Folder layout

```
Documents\Claude\
  CLAUDE.md               — playbook (read this)
  about-me.md             — Garman's background and situation
  how-to-speak.md         — tone and style rules
  memory.md               — in GitHub\ subfolder, durable facts
  github-deploy.md        — deploy token and rules
  api.txt                 — Garman's own copy of API tokens
  1_Inbox\                — Garman drops task files here
  2_Working\              — tasks in progress
  3_Results\              — delivered outputs
  4_Archive\              — completed task files
  _Templates\             — task template, daily briefing template
  GitHub\
    agent-dashboard\      — dashboard repo local clone
      index.html          — THE dashboard (edit this, never duplicate)
      docs\               — jobs txt files served by GitHub Pages
    memory.md             — cross-session memory
  Job Search\
    Job Searches\         — jobs_[date].txt files, application docs
    Garman_Yip_Master_Profile.md — canonical resume/scoring doc
```

---

## Known gotchas

- `_Profile/` folder is referenced in CLAUDE.md but was never created. The profile files (`about-me.md`, `how-to-speak.md`) live at the root of `Documents\Claude\`. `memory.md` is in `GitHub\`.
- The bash sandbox view of OneDrive files can be stale or truncated. Use the Read/Write file tools as authoritative, not bash cat/ls when the content matters.
- Dashboard `"enabled"` flag in the embedded JSON: if false, skip dashboard updates until Garman toggles it back on.
- Token figures in the dashboard are estimates, not billing data.
- Portfolio site (`gyip.com`, repo: `delreyla-max.github.io`) — Claude has write access via the same token but NEVER pushes without Garman's explicit instruction.
