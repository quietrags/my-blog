---
title: "Claude Code vs Agent SDK: When Prompts Meet Code"
date: 2024-12-24 10:00:00 +0530
categories: [AI Agents, Development]
tags: [claude-code, agent-sdk, architecture, orchestration, agentic-systems]
excerpt: "I spent weeks building agentic workflows in Claude Code before I understood what the Agent SDK actually gives you. Here's what I learned about choosing between prompt-based and code-based orchestration."
toc: true
---

## The Question That Started It All

Last week, I was trying to build a multi-stage command in Claude Code. The pattern seemed simple enough: ask the user a question, call an LLM to process their answer, hand it off to an agent, and have that agent use a skill to finish the job. Four stages. Clean pipeline.

But then I wondered: what if I built this same thing with the Agent SDK instead? Would that even make sense? Are they different tools for different jobs, or just two ways to do the same thing?

I didn't have a clear answer. So I dug in.

## The Architecture Nobody Explains

Here's what I learned: the difference between Claude Code and Agent SDK isn't about capabilities—it's about **who controls the flow**.

### Claude Code: Prompts as Orchestration

In Claude Code, you write markdown prompts that Claude interprets. The "logic" is implicit—Claude reads your instructions and decides which tools to call and when.

The architecture looks like this:

```
┌─────────────────────────────────────┐
│     Slash Command (prompt)          │
│     .claude/commands/foo.md         │
└──────────────┬──────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│     Claude (Main) Interprets        │
│     Decides tool order              │
└──────────────┬──────────────────────┘
               │
               ▼
     AskUser → Bash(llm) → Task(agent) → Skill
```

You write a markdown file like this:

```markdown
# .claude/commands/my-pipeline.md

You are executing a multi-stage pipeline. Follow these steps:

## Stage 1: Gather User Input
Use AskUserQuestion to ask: "What would you like to do?"

## Stage 2: External LLM Processing
Run: `llm -m claude-3-haiku "Process this: {user_input}"`

## Stage 3: Agent Delegation
Use Task tool with the LLM response.

## Stage 4: Skill Completion
The agent should invoke skill "format-output".
```

Claude reads that, interprets it, and figures out the execution path. Control is **implicit**—you describe what should happen, and Claude decides how.

### Agent SDK: Code as Orchestration

With the Agent SDK, you write Python or TypeScript. **You** control the flow explicitly. Agents become objects you instantiate. Skills become tool functions. There's no interpretation layer—just code calling agents when you tell it to.

Here's the same pipeline in Agent SDK:

```python
from anthropic_agent_sdk import Agent

# Stage 1: User Input (your code controls this)
user_input = input("What would you like to do? ")

# Stage 2: LLM Processing (agent in query mode)
analyzer = Agent(
    model="claude-sonnet-4-20250514",
    system="You analyze requests and extract requirements.",
    tools=[]  # No tools = just an LLM call
)
analysis = analyzer.query(f"Analyze: {user_input}")

# Stage 3: Agent Delegation (agent with tools)
executor = Agent(
    model="claude-sonnet-4-20250514",
    system="You execute tasks using available tools.",
    tools=[skill_tool_a, skill_tool_b]
)
result = executor.run(f"Execute based on: {analysis}")

print(result)
```

Control is **explicit**—you decide when to call each agent, what to pass between them, and when to stop.

## Mapping the Concepts

When I first tried to map Claude Code concepts to Agent SDK, I got confused. Commands, agents, skills, scripts—how do these translate?

Here's what clicked for me:

| Claude Code | What It Really Is | Agent SDK Equivalent |
|-------------|-------------------|---------------------|
| **Command** (`.claude/commands/foo.md`) | Entry point as a prompt | `def main():` or CLI entry point |
| **Agent** (`Task(subagent_type="...")`) | Subprocess with tools | `Agent(tools=[...])` instance |
| **Skill** (`.claude/skills/bar.md`) | Specialized knowledge | `@tool` decorated function |
| **Script** (`~/bin/my-script`) | External LLM call | `agent.query()` or direct API |

The key insight: **in Claude Code, everything is a prompt**. Commands are prompts. Skills are prompts. Even agents are just Claude interpreting prompts in a subprocess.

In Agent SDK, **everything is code**. Commands are functions. Skills are callable tools. Agents are objects you control.

## The Spectrum: LLM Call to Agent Harness

One question kept nagging me: if I use the Agent SDK just to make a single LLM call, is that still an "agent"?

Turns out, it's a spectrum:

```
┌──────────────────────────────────────────────────────────────┐
│                                                              │
│   Raw LLM Call     Agent SDK (Query)     Agent SDK (Loop)   │
│   ─────────────    ────────────────      ───────────────    │
│                                                              │
│   • Single turn    • Single turn OR      • Autonomous loop  │
│   • No tools       • Tools available     • Tools + decides  │
│   • You parse      • Structured output   • Runs until done  │
│   • Stateless      • Can be stateless    • Maintains state  │
│                                                              │
│   ◄──── LLM Call ────────►◄──── Agent Harness ──────►       │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

The Agent SDK **provides** an agentic loop, but doesn't **force** you to use it. You can call `agent.query()` without tools and get a one-shot LLM response. That's just an LLM call with SDK overhead.

Or you can call `agent.run()` with tools and let it loop autonomously until the task is done. That's a true agent harness.

I learned this the hard way when I built my first "agent" that was really just a fancy API wrapper. It worked, but I wasn't using the SDK's strengths.

## When to Use Which

After building pipelines both ways, here's my decision heuristic:

**Use Claude Code when:**
- You want natural language orchestration (prompts are easier to write than code)
- The workflow might change frequently (editing markdown is faster than refactoring code)
- You're working inside Claude Code already and don't need external deployment
- You want Claude to decide the tool execution order (implicit control is actually a feature)

**Use Agent SDK when:**
- You need precise control over execution flow (explicit beats implicit)
- You're building a deployment that runs outside Claude Code (CLI tool, web service)
- You want to version control your agent logic with standard dev tools (git, tests, CI/CD)
- You need complex error handling or retry logic that's easier to express in code

I used to think Agent SDK was just "Claude Code but in Python." Wrong. Agent SDK is for when you **don't want Claude interpreting prompts**—when you want to write the orchestration logic yourself and use agents as reasoning components you control.

## The Fundamental Shift

Here's the sentence that changed how I think about this:

**In Claude Code, you write prompts that Claude interprets to decide control flow. In Agent SDK, you write code that explicitly controls flow and uses agents where you need reasoning.**

Skills become tools. Agents become objects. Commands become functions. And you're the orchestrator, not Claude.

For my multi-stage pipeline? I ended up using Claude Code. Why? Because the complexity was in the **what** (what stages to run), not the **how** (how to execute them). I wanted to describe the workflow and let Claude figure out the details.

But for a production deployment where I need guaranteed retry behavior, structured logging, and integration with existing Python services? Agent SDK all the way.

## What Still Confuses Me

I'm still figuring out the boundary cases. Like:
- When does a complex Claude Code command become too implicit to maintain?
- When should I use nested agents (agents spawning agents) instead of writing explicit orchestration code?
- How do I test Agent SDK workflows effectively when the agent makes autonomous decisions?

If you've solved these, I'd love to hear about it.

## The Takeaway

The choice isn't about which is "better." It's about matching the tool to your control requirements.

Prompts for orchestration when you want Claude deciding. Code for orchestration when you want explicit control. Both can build the same pipeline. The difference is who's driving.

And that difference matters more than I expected.
