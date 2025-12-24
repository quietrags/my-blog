---
title: "Managing Multi-Session Workflows in Claude Code"
date: 2024-12-24 10:00:00 +0530
categories: [Development, AI Agents]
tags: [claude-code, workflow, productivity, beads, session-management]
excerpt: "I used to lose my work context every time I closed Claude Code. Plans vanished, decisions got re-litigated, and I'd spend the first 20 minutes of each session just trying to remember where I left off. Then I discovered the three-file system, and everything changed."
toc: true
---

## The Context Loss Problem

Have you ever started a coding session, spent an hour planning the perfect architecture, gotten approval from your team, and then... life happened? You close the session, and when you come back three days later, that entire plan exists only in your chat history—if you're lucky enough to still have the same conversation thread open.

I hit this wall hard about two months ago. I was building a research pipeline with Claude Code, had mapped out this elegant seven-phase plan, and got so excited that I immediately dove into Phase 1. Finished it, felt great, closed my laptop for the weekend. Monday morning? I stared at my screen for 15 minutes trying to reconstruct what Phases 2 through 7 even were.

That's when I learned about multi-session workflows the hard way.

## The Three-File System: My North Star

The breakthrough came when I stopped treating Claude Code sessions like they were continuous and started treating them like what they actually are: discrete conversations that need explicit handoffs.

I discovered—or rather, stumbled into—a three-file system that changed everything:

### CLAUDE.md: The Project DNA

This file answers "How does this project work?" It's permanent truth. Architecture decisions, coding conventions, tool configurations. When I refactor something fundamental, CLAUDE.md gets updated to reflect the **current state**, not the history.

I learned this distinction the hard way when I documented a REST API design in CLAUDE.md, then three sessions later switched to WebSockets. For two weeks, CLAUDE.md still said "We use REST endpoints" and I kept getting suggestions that contradicted my actual implementation.

Now? CLAUDE.md shows current truth only. Old approaches get replaced, not archived.

### progress.md: The Learning Journal

Here's what surprised me: CLAUDE.md tells you WHAT, but it doesn't tell you WHY. That's progress.md's job.

Every time I end a session where I learned something, discovered an anti-pattern, or made a key decision, I run `/log-session`. It captures:
- What I tried
- What failed and why
- What worked
- What I'd do differently next time

The format I've settled on looks like this:

```markdown
## Session: 2024-12-15

### Decisions
- Switched from polling to WebSocket streaming
- Rationale: Real-time features needed push, not pull

### Learnings
- Query on user_sessions was slow
- Profiling showed missing index on user_id
- Adding index cut query time from 450ms to 12ms

### Blockers
- Can't test WebSocket reconnection locally
- Need staging environment access
```

progress.md has saved me countless hours of re-learning the same lessons.

### Beads: The Task Tracker

This is where plans actually persist. I use `bd` (Beads CLI) to track:
- Work items across phases
- Status (open/in_progress/blocked/done)
- Dependencies between tasks
- What's ready to work on next

The golden rule I learned? **Never start multi-phase work without persisting the plan to Beads first.**

Here's the workflow that finally stuck:
1. Research and plan (in conversation)
2. Get approval
3. **Persist to Beads** (`bd create` for each task)
4. Execute

That third step? That's the one I kept skipping. And every time I skipped it, I paid for it later.

## Time Horizons: What Survives When

I spent three sessions confused about when to use which file until I mapped out the time horizons:

| File | Time Horizon | Survives | Example |
|------|-------------|----------|---------|
| **CLAUDE.md** | Permanent | Project lifetime | "We use JWT tokens with 24h expiry" |
| **progress.md** | Session-to-session | Git history | "2024-12-15: Chose JWT over sessions because..." |
| **Beads** | Task lifecycle | Until task closes | "bd-15: Implement token refresh (in_progress)" |

The key insight? **Beads tracks tasks, not knowledge.** When a task closes, the learning disappears unless you've captured it in CLAUDE.md or progress.md.

I lost an entire week's worth of debugging insights because I marked `bd-23` as done and never documented what I'd learned about database connection pooling. Had to re-discover it all two months later.

## The Information Lifecycle

Here's the pattern I've developed for when I discover something important:

**Discovery happens → Log in progress.md**
- Captures the WHY and WHEN
- Example: "2024-12-18: Queries slow on user_sessions. Profiling showed missing index."

**Becomes permanent truth? → Add to CLAUDE.md**
- Captures the WHAT (current state)
- Example: "Database: user_sessions requires index on user_id for session lookup performance"

**Work is needed? → Create Beads task**
- Captures the TODO
- Example: "bd-15: Add index on user_sessions.user_id" (status: done)

All three capture different aspects of the **same event**. That redundancy isn't waste—it's insurance against context loss.

## Handling Refactors and Evolution

I'm three sessions into a project. The API design from session 1 was wrong. I've refactored to a better approach. Now what?

**CLAUDE.md gets updated** to reflect current truth. I replace the old approach with the new one. No history preserved.

**progress.md gets the evolution story.** I log what changed and WHY:

```markdown
## Session: 2024-12-22

### Decisions
- Refactored from REST to WebSocket
- Rationale: Real-time features needed push, not pull

### What Changed in CLAUDE.md
- Old: /api/events (polling)
- New: ws://api/stream (push)
```

This distinction took me four refactors to internalize. CLAUDE.md is a snapshot. progress.md is a film strip.

## Session Handoffs: The Blocked State

I hit a blocker last Thursday. Can't test WebSocket reconnection without staging access, don't have credentials yet. What do I do before ending the session?

Two steps, both critical:

1. `bd update bd-34 --status blocked`
2. `/log-session` to capture what I tried, why it failed, and ideas for next steps

When I return on Monday, I run:
- `bd ready` → Shows "bd-34: blocked"
- Read progress.md → Shows full context of what I tried and why

Without both pieces, I'd waste Monday morning re-discovering the blocker.

**Why both?** Beads tells WHAT is blocked. progress.md tells WHY and what was already attempted. You need both to resume effectively.

## Fresh Project Setup: Order Matters

When starting a new project, I've learned the setup order matters:

**Step 1: Create CLAUDE.md**
- Document what exists (if existing project)
- Establish conventions (if new project)

Minimum viable CLAUDE.md:
```markdown
# Project Name

## Project Structure
src/           # Source code
tests/         # Test files
docs/          # Documentation

## Development Conventions
- TDD: Write tests first
- TypeScript strict mode
- Prettier for formatting

## Discovered Patterns
(To be filled as you learn)
```

**Step 2: Initialize Beads**
```bash
bd init --quiet
```

**Step 3: Create progress.md**
- On first meaningful session via `/log-session`

I used to do these in random order. That created confusion about where to document what.

## The Onboarding Sequence

Here's what blew my mind: **the same onboarding pattern works for both human contributors AND new Claude Code sessions.**

When someone asks "How do I get up to speed?" the answer is:

1. **Read CLAUDE.md** → Understand the project (WHAT)
2. **Read progress.md** → Understand the evolution (WHY)
3. **Check Beads** → Find available work (WHAT'S NEXT)

This gives full context before diving in. No one makes decisions that contradict past learnings or architectural choices.

I onboarded a colleague last week using this exact sequence. Took him 30 minutes to read through everything, then he picked up `bd-42` and shipped it that afternoon. No questions about "why did we choose X" or "what was the rationale for Y"—it was all already documented.

## Common Misconceptions (That I Definitely Had)

**Misconception #1: "Beads handles all my context"**

Reality: Beads tracks tasks, not knowledge. When a task closes, the context disappears unless captured elsewhere.

**Misconception #2: "Task done = no documentation needed"**

Reality: The learning disappears when the task closes. If it's worth knowing, it's worth documenting in CLAUDE.md or progress.md.

**Misconception #3: "Plans in conversation persist"**

Reality: Conversation context does NOT survive session boundaries. Must persist to Beads before execution starts.

I violated all three of these for weeks. Paid for it every time.

## What I'm Still Figuring Out

I'm not certain this is optimal. There's friction in the handoff process—remembering to run `/log-session` before closing, making sure Beads tasks are granular enough but not too granular, deciding what deserves documentation vs. what's just ephemeral context.

Some days I wonder if I'm over-documenting. Other days I'm grateful Past Anurag left detailed notes.

The balance is still evolving. If you've found a better system for managing multi-session work, I'd genuinely love to hear about it.

## Key Takeaways

After eight weeks of refining this system:

1. **Three files, three purposes**: CLAUDE.md (how), progress.md (why), Beads (what's next)
2. **Never start multi-phase work without persisting to Beads**—learned this one the expensive way
3. **Session handoffs need both status (Beads) AND context (progress.md)**—one without the other is incomplete
4. **Same onboarding sequence works for humans AND Claude sessions**—read CLAUDE.md → progress.md → Beads

The system isn't perfect. But it's turned multi-session work from a context-losing nightmare into something actually manageable.

And I haven't lost a plan to session boundaries in six weeks. That alone makes it worth it.
