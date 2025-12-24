---
title: "Understanding How Claude Code Really Works"
date: 2024-12-24 10:00:00 +0530
categories: [AI Agents, Development]
tags: [claude-code, agentic-coding, workflow, learning, mental-models]
excerpt: "I thought I understood Claude Code's architecture until one question revealed I had the entire execution model backwards. Here's what I learned through a Socratic deep-dive into how it actually orchestrates tools, context, and agents."
toc: true
---

## The Moment I Realized I Was Wrong

I'd been using Claude Code for weeks. I could name every tool—Bash, Read, Write, Fetch. I knew about the 200k context window. I understood sub-agents had separate contexts. My vocabulary was solid.

Then someone asked me: "When Claude Code calls a tool like Read or Bash, what happens to the conversation flow?"

My answer was completely wrong.

I thought the tool executed, the result got appended, and the same turn continued. Seemed logical, right? Turns out, I had fundamentally misunderstood how agentic coding actually works. That one question unraveled my entire mental model.

## What I Got Wrong About Tool Execution

Here's what I believed happened:

```
User asks → Claude thinks → Calls tool → Tool runs →
Result appears → Claude continues thinking
```

All in one smooth flow. One API call. One turn.

**That's not how it works at all.**

Here's what actually happens:

```
User Message → API Call #1 → Claude generates tool_call block →
Claude STOPS → CLI executes tool → API Call #2 with tool_result →
Claude continues
```

Each tool call is a **complete round-trip**. Claude doesn't "pause and resume"—it stops completely. The CLI executes the tool locally. Then a fresh API request carries the result back to Claude.

This blew my mind. How had I been using this system for weeks without understanding this fundamental behavior?

## Why This Misconception Matters

Once I understood the round-trip model, suddenly everything else made sense.

**Why does Claude Code support calling multiple tools in parallel?**

Before I understood the loop structure, I couldn't answer this. After the correction, it clicked immediately:

1. **API Optimization**: If you need 3 file reads, one generation with 3 tool calls means 1 round-trip instead of 3
2. **Execution Optimization**: Claude Code can run all 3 tools concurrently before the next API call

What looked like a convenience feature was actually a critical performance optimization. Each avoided round-trip saves latency and keeps the conversation flowing.

## The Context Window Problem I Wasn't Thinking About

Next question that caught me off-guard: "The 200k context window fills up with messages and tool results. What happens when a conversation approaches this limit?"

I knew the window existed. I knew it could fill up. But I'd never thought through what happens *when* it fills up.

**The answer: Automatic summarization.**

Claude Code doesn't just crash or warn you. It starts compressing older exchanges, preserving key decisions and context, allowing sessions to extend "indefinitely."

Sounds great, right? Unlimited session length?

Not so fast.

## What Summarization Actually Costs You

Here's the scenario that made this concrete for me:

*You're 2 hours into a complex refactoring session. You mentioned a specific design decision early on—maybe which architectural pattern to follow, or why you chose a particular data structure. What's the risk?*

The early decision could be **summarized away** as context compresses.

I've experienced this. I've been deep in a session and had Claude forget something I mentioned an hour ago. I thought it was a bug or a limitation of the model. Turns out, it's the trade-off of summarization.

**What gets lost:**
- Early details lose fidelity
- Specific file paths might blur
- Exact wording of decisions

**The mitigation I learned:**
- Use TodoWrite to persist decisions in the active context
- Update CLAUDE.md with key architectural choices
- Reinforce critical information periodically

Long sessions aren't "free." Summarization extends length but trades off recall precision.

## The Real Test: A 4-Hour Coding Session

The final challenge question was the hardest:

*Design a strategy for a 4-hour coding session where you need to maintain 3 key architectural decisions throughout. What combination of Claude Code features would you use?*

Before understanding these concepts, I would have just... talked more clearly? Repeated myself? Maybe written things in comments?

After understanding the execution loop, context management, and summarization, I realized you need a **multi-layer persistence strategy**:

### Layer 1: Human-Level (Beads)
For project-level decisions that span multiple sessions. Beads issues persist across conversations and give you cross-session memory.

### Layer 2: Session-Level (CLAUDE.md + progress.md)
For architectural decisions that need to survive summarization within a session. These files get read at session start and provide grounding context.

### Layer 3: Task-Level (TodoWrite)
For phase breakdowns and in-context tracking during active work. Keeps current tasks visible without relying on conversation history.

### Layer 4: Execution (Sub-agents)
For complex tasks that need fresh context windows. You pass decision context explicitly rather than hoping it survives in the parent context.

Each layer has a different **persistence lifetime** and **access cost**. The strategy is matching the right tool to the need.

## What "Understanding" Actually Means

I started this learning session thinking I understood Claude Code because I could name its parts.

What I learned is that vocabulary and mental models are completely different things.

- **Vocabulary**: I could name tools, context windows, sub-agents—that was solid from day one
- **Mental Model**: I had the execution flow backwards, didn't understand the costs of summarization, couldn't reason about optimization trade-offs

The Socratic questioning method revealed this gap brutally. Each question forced me to predict behavior based on my model. When my predictions were wrong, I had to rebuild the model from scratch.

**Before:** Vocabulary solid, mental model misconceived
**After:** All facets solid, could transfer understanding to new scenarios

I passed two transfer checks—questions I hadn't seen before, requiring me to apply the corrected model to novel situations. That's when I knew I'd actually learned it.

## The Questions That Changed My Understanding

Looking back, three diagnostic questions did all the work:

1. **"When Claude Code calls a tool, what happens to the conversation flow?"**
   Revealed I didn't understand the round-trip execution model

2. **"Why does Claude Code support calling multiple tools in parallel?"**
   Forced me to reason from the execution model to optimization benefits

3. **"What happens when a conversation approaches the context limit?"**
   Exposed that I hadn't thought through the consequences of long sessions

Each question was carefully chosen to reveal a specific gap. Not trivia. Not testing recall. Testing whether my mental model could predict actual system behavior.

## What I'm Still Figuring Out

I'm not certain this multi-layer persistence strategy is optimal. It's what made sense after understanding the architecture, but I've only used it for a few sessions.

There might be simpler patterns. There might be cases where Beads is overkill and TodoWrite is enough. I'm experimenting.

If you've found different strategies that work better for complex multi-session projects, I'd genuinely love to hear about them.

## Why This Matters for Agentic Coding

Have you ever watched a Claude session eat through your context window and wondered where it all went? That's the token problem with traditional MCP.

Understanding the execution loop helps you see why:
- Each tool result consumes context
- Multiple round-trips stack up the conversation history
- Summarization compresses old content but loses precision

Think of it like this: traditional tool calling is like someone reading you an entire phone book to find one name. The whole book enters the conversation.

Understanding the round-trip model helps you optimize for the *right* thing—not just making Claude work, but making it work *efficiently* within the constraints that actually exist.

## The Takeaway

Each tool call is a round-trip.
Context grows with every exchange.
Summarization compresses but doesn't preserve everything.
Persist what matters across the right layers.

That's Claude Code's architecture in four sentences. It took me hours of Socratic questioning to rebuild my mental model from "vocabulary solid" to "can design 4-hour session strategies."

But now when I use Claude Code, I'm not just running commands. I'm reasoning about round-trips, managing context strategically, and choosing the right persistence layer for each decision.

That's what understanding actually feels like.

---

*What surprised me most about this learning process was how confidently wrong I was at the start. Strong vocabulary creates the illusion of understanding. Testing the mental model reveals the truth.*
