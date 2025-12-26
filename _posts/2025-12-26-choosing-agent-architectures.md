---
title: "Choosing Agent Architectures: A Practitioner's Decision Framework"
date: 2025-12-26 22:30:00 +0530
categories: [AI Agents]
tags: [react, reflexion, plan-and-execute, agent-architecture, decision-framework]
toc: true
excerpt: "The key insight isn't which architecture is 'best'—it's matching the architecture to the specific problem you're solving."
---

The conventional framing of agent architectures—ReACT vs. Reflexion vs. Plan-and-Execute—suggests you need to pick one and commit. What I've come to appreciate is that this framing obscures the more useful question: what specific problem are you trying to solve?

## The Problem-Architecture Mapping

After working through various agent implementations, a clearer decision framework has emerged. The question isn't "which architecture is most sophisticated?" but rather "what's going wrong with my current agent?"

- **Structure problems → Planning.** If your agent produces incoherent multi-step outputs, if steps depend on each other but lack coordination, the solution is deliberative planning—not more reactive loops.

- **Quality problems → Reflection.** If your agent's individual outputs are inconsistent or error-prone, adding self-critique (Reflexion) addresses this directly. Planning won't help here.

- **Speed/cost problems → Simplify.** If latency or API costs are the constraint, strip back to basic ReACT. The overhead of planning and reflection may not be justified.

This framing has been more useful than the typical "complexity ladder" that implies you should always be climbing toward more sophisticated architectures.

## What I Initially Misunderstood About ReACT

My initial assumption was that when a ReACT agent fails, the architecture itself is the problem—time to add planning or reflection. What I've found is that most "ReACT failures" are actually configuration issues.

Consider a failure trace where the agent calls the same tool repeatedly with no progress. The instinct is to conclude that ReACT can't handle this task. But analyzing the trace often reveals simpler causes: the tool description is ambiguous, the agent lacks an explicit "done" signal, or context is overflowing and the agent is losing track of its goal.

The practical implication: before switching architectures, analyze the failure trace. The fix is often a better prompt, clearer tool descriptions, or explicit guardrails—not a wholesale architecture change.

### ReACT's Real Failure Modes

That said, ReACT does have genuine limitations worth understanding:

| Failure Mode | Symptom | Mitigation |
|--------------|---------|------------|
| Infinite loops | Agent repeats actions without progress | Max iterations, token budgets, explicit termination tools |
| Context overflow | Agent "forgets" earlier context | Scratchpad pattern, summarization, working memory |
| Tool fixation | Over-reliance on one tool despite alternatives | Tool diversity prompting, tool recommendation logic |
| No stopping criteria | Agent doesn't know when "good enough" is achieved | Explicit "task_complete" tool, success criteria in prompt |

The key insight here is that ReACT has no internal sense of "done." It will keep iterating until something external stops it. This isn't a bug—it's fundamental to how reactive architectures work. External termination logic isn't optional; it's required.

## The Complexity Cost

A second misconception I've revised: the assumption that adding planning and reflection always improves outcomes.

Each architectural layer adds cost:
- **Latency:** Multiple LLM calls per user request
- **API costs:** Reflexion roughly doubles or triples token usage
- **Debugging surface:** More components means more places things can break
- **Maintenance burden:** More sophisticated systems require more expertise to operate

For a system handling 100K queries per month, adding Reflexion at 2-3x the token cost is a significant decision. The quality improvement needs to justify that expense.

The principle I've adopted: start with ReACT, add complexity only when you've proven you need it. If ReACT with good prompts and guardrails handles 90% of cases well, that remaining 10% might not justify doubling your architecture's complexity.

## When Reflexion Actually Helps

Reflexion isn't universally beneficial—it's specifically a quality-for-time trade. The architecture works by having the agent critique its own output, identify issues, and revise.

This works well when:
- Quality variance is the problem (some outputs good, some poor)
- You have meaningful evaluation criteria (objective or well-defined subjective)
- Extra latency and cost are acceptable
- Tasks are generative (writing, coding, design)

It fails when:
- Speed is critical (the critique loop adds latency)
- No meaningful quality signal exists (garbage-in, garbage-out on evaluation)
- Volume is high and stakes are low (the per-request cost doesn't justify itself)

One nuance I initially missed: for code generation, external signals like tests and linters are more reliable than LLM self-judgment. The agent doesn't need to evaluate its own code—it can run tests. This shifts Reflexion from "self-critique" to "external feedback integration," which is more robust.

## The Replanning Question

For Plan-and-Execute architectures, a key design decision is whether to allow replanning when execution hits obstacles.

My initial assumption was that replanning makes agents more adaptive—surely flexibility is good? What I've come to appreciate is that this depends heavily on stakes and reversibility.

For a deployment pipeline, "creative replanning" that decides to skip tests is catastrophic. For customer support ticket routing, adapting to unexpected input is valuable.

The framework I use now:
- **High stakes, irreversible actions → Lock the plan.** Escalate to humans on failure rather than attempting recovery.
- **Lower stakes, reversible actions → Allow replanning.** Adaptation improves outcomes.

The key question: "If the agent's replanning goes wrong, what's the worst case?" If the worst case is bad (financial loss, data corruption, security breach), lock the plan.

## Hybrid Architectures

The architectures aren't mutually exclusive. You can stack them based on specific needs:

- **Plan + ReACT substeps:** High-level plan provides structure, ReACT handles individual step execution with flexibility.
- **ReACT + Reflexion:** Reactive execution with quality checking on outputs.
- **Plan + Reflexion:** For tasks like email campaigns where you need both sequence coherence (planning) AND quality per message (reflection).

The principle is the same: identify the specific problem, add the targeted solution. Don't stack architectures speculatively—each addition should address a demonstrated gap.

## Open Questions

Several aspects I haven't fully resolved:

- **Memory at scale:** How do production systems handle cross-task memory with millions of interactions? The session-based patterns I've used don't obviously scale.

- **Evaluation calibration:** Systematic methods for tuning Reflexion evaluators. How harsh should self-critique be? Too harsh creates infinite loops; too lenient provides no quality improvement.

- **Cost modeling:** Building a formal model for architecture decisions. At what quality improvement threshold does Reflexion's 2-3x cost increase become worthwhile?

These remain areas for deeper exploration.

## The Decision Framework

To summarize the practical framework:

1. **Start simple.** Begin with ReACT plus good prompts and guardrails.

2. **Diagnose specific failures.** When things go wrong, identify the category: structure problem, quality problem, or speed/cost problem.

3. **Add targeted solutions.** Structure issues → planning. Quality issues → reflection. Don't add complexity you haven't proven you need.

4. **Consider stakes for replanning.** High stakes and irreversibility → locked plans with human escalation. Lower stakes → adaptive replanning.

5. **Stack deliberately.** Hybrid architectures solve multiple problems but compound costs. Each layer should address a demonstrated gap.

The key insight isn't which architecture is "best"—it's that architecture choices are problem-specific. The question "should I use ReACT or Plan-and-Execute?" is incomplete without understanding what's currently failing and why.

---

*Author: Anurag Sahay, Group CTO, Data & AI | Nagarro*
