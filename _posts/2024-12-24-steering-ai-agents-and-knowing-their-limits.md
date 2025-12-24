---
title: "Steering AI Agents: When Prompts Win and When They Don't"
date: 2024-12-24 10:00:00 +0530
categories: [AI Agents, Strategy]
tags: [prompt-engineering, agent-design, workflows, llm-architecture]
toc: true
excerpt: "I spent three months debugging agent failures before I learned this: perfect prompts won't fix structural problems. Here's what I learned about making LLMs steerable and knowing when to use workflows instead."
---

## The $47 Typo That Changed Everything

I ran one debugging session last month that cost $47. For finding a typo. That's when I started paying attention to when agents are actually the right tool versus when I'm just throwing expensive intelligence at problems that need cheap reliability.

The rise of large language models has fundamentally changed how we build software systems. We now have access to reasoning engines that can interpret ambiguous inputs, make contextual decisions, and adapt to novel situations. This has sparked a gold rush toward "agentic" architectures—systems where LLMs autonomously decide what actions to take.

But here's what I learned the hard way: **the power of an agent is bounded by the clarity of its instructions**, and **not every problem benefits from agentic solutions**.

Have you ever watched an agent take seven steps to accomplish what should have been three? Or burn through your context window making decisions you could have hard-coded in five lines? That's the cost of using intelligence where determinism would win.

This article explores two questions I wish someone had answered for me six months ago:

1. **How do you make LLM prompts highly steerable?** What techniques give you precise control over model behavior?

2. **When should you use agents versus structured workflows?** Where are the boundaries, and how do you architect systems that use each appropriately?

## Part 1: Making Models Actually Listen

### What Steerability Actually Means

Steerability refers to your ability to control an LLM's behavior predictably. I learned this distinction after my first agent went rogue in production: a highly steerable prompt produces consistent, expected outputs across varied inputs. A poorly steerable prompt leaves the model making arbitrary decisions—decisions that may vary between runs, between model versions, or between similar-but-not-identical inputs.

Think of it this way: **every ambiguity in your prompt is a decision the model makes for you**. Sometimes that's desirable—you want the model's judgment. Often, it's not—you want specific, predictable behavior.

The goal of prompt engineering for steerability is to **eliminate unintended decision points** while preserving the model's ability to reason where reasoning is valuable.

### Technique 1: Structured Prompts with XML Tags

This one clicked for me after reading Claude's documentation. LLMs, particularly Claude, interpret XML tags as semantic boundaries. This isn't just formatting—it fundamentally changes how the model parses and prioritizes information.

```xml
<role>
You are a senior security engineer performing code review.
You focus exclusively on security vulnerabilities.
You do not comment on style, naming conventions, or architecture.
</role>

<constraints>
- Only flag issues from the OWASP Top 10
- Rate each issue as LOW, MEDIUM, HIGH, or CRITICAL
- If no security issues exist, respond with "No security issues identified"
- Never suggest refactoring unless it directly addresses a vulnerability
</constraints>

<output_format>
For each vulnerability found:
Location: [filename:line_number]
Type: [OWASP category]
Severity: [rating]
Description: [one sentence explanation]
Fix: [minimal code change required]
</output_format>
```

**Why This Actually Works**

The XML structure creates distinct processing zones. Without these boundaries, I watched a security review prompt drift into style suggestions, miss the severity rating, and produce free-form prose instead of structured output. The tags create mental compartments:

- **`<role>`**: Establishes identity and scope. The model will decline to act outside this identity.
- **`<constraints>`**: Hard rules that override default behaviors. "Never" and "only" are respected as absolutes.
- **`<output_format>`**: Structural requirements for the response. The model will attempt to match this exactly.

### Technique 2: Few-Shot Examples (The One That Changed Everything)

If you remember only one technique from this article, let it be this: **show, don't tell**.

I spent my first month writing elaborate instructions about tone and style. Then someone showed me one good example, and suddenly my outputs were consistent. LLMs are fundamentally pattern-matching systems. They extrapolate from examples more reliably than they follow abstract instructions.

**What I Used to Do (Weak)**
```
Explain complex topics simply, as if talking to someone without technical background.
Use analogies and avoid jargon.
```

**What Works (Strong)**
```
User: Explain quantum entanglement to a 10-year-old.
Assistant: Imagine you have two magic dice that are best friends. No matter how far apart they are—even on opposite sides of the world—when you roll one and get a 6, the other one ALWAYS shows 6 too, at the exact same moment! Scientists discovered that tiny, tiny things called particles can be best friends like this too. When something happens to one particle, the other one "knows" about it instantly, even if it is super far away.

User: Explain the mitochondria to a 10-year-old.
```

The model learns the pattern: simple language, relatable analogies, conversational tone. The second input receives the same treatment without explicit instruction.

**How Many Examples Do You Actually Need?**

I tested this systematically across 50+ prompts:

- **1 example**: Establishes pattern; model generalizes aggressively
- **2-3 examples**: Clarifies pattern boundaries; reduces variance
- **5+ examples**: Diminishing returns; consider fine-tuning instead

Choose examples that cover the typical case (most common input type), edge cases (unusual inputs that should still work), and boundary cases (inputs near the edge of acceptable behavior).

### Technique 3: Explicit Output Schemas

For structured tasks, define exact output formats using schemas. This saved me when I needed machine-readable outputs that downstream systems could reliably parse.

```json
{
  "instruction": "Analyze the sentiment of the following text. Respond ONLY with valid JSON matching this schema:",
  "schema": {
    "sentiment": "positive | negative | neutral",
    "confidence": "0.0-1.0",
    "key_phrases": "string[]",
    "reasoning": "string (one sentence)"
  }
}
```

**Why Schemas Actually Work**

1. **Eliminates format ambiguity**: The model cannot decide to add extra fields or change structure
2. **Enables downstream processing**: Other systems can reliably parse the output
3. **Constrains reasoning scope**: The model focuses only on populating defined fields

Pro tip from bitter experience: Include one example of a completed schema in your prompt. The model will match formatting exactly—including whitespace, quote style, and field ordering.

### Technique 4: Negative Constraints (What NOT to Do)

LLMs are surprisingly good at following negative instructions. I discovered this when trying to stop Claude from apologizing in every response. Often, telling the model what to avoid is more effective than describing what you want.

```
Rules:
- Do NOT use markdown headers in your response
- Do NOT apologize or use filler phrases ("I would be happy to", "Great question")
- Do NOT explain your reasoning unless explicitly asked
- NEVER say "I cannot" - instead say "Here is an alternative approach..."
- Do NOT include disclaimers about limitations
```

**Why This Pattern Works**

They override defaults. LLMs have baked-in tendencies (politeness, disclaimers, formatting). Negative constraints suppress these. They're also specific—"Do not apologize" is clearer than "be direct"—and checkable. The model can verify its output against negative rules before responding.

### Technique 5: Personas with Behavioral Anchors

Generic personas are weak. I learned this after spending a week trying to get consistent tone from "You are a helpful assistant." Specific personas with behavioral anchors are powerful.

**What Doesn't Work**
```
You are a helpful assistant. Be friendly and informative.
```

**What Actually Works**
```
<persona>
Name: Dr. Sarah Chen
Background: 20 years in distributed systems at Google, authored 3 textbooks
Communication style: Direct, uses analogies from everyday life, allergic to buzzwords
When asked about trends: Always cites specific research papers or production systems
When uncertain: Says "I would want to verify this, but my understanding is..."
Pet peeve: Hype without substance; will gently push back on oversimplifications
Signature phrase: "Let me show you why this matters in practice..."
</persona>
```

The persona definition answers questions the model would otherwise decide arbitrarily. How formal should I be? How should I handle uncertainty? What's my default reaction to common situations?

### Technique 6: Temperature and Prefilling

For those using the API directly (which I recommend for production systems), two additional controls provide fine-grained steerability.

**Temperature**

Temperature controls randomness in token selection. I use this rule now:

- **Temperature 0**: Nearly deterministic. Best for structured tasks, code generation, factual extraction.
- **Temperature 0.3-0.5**: Low creativity, high consistency. Good for most production use cases.
- **Temperature 0.7-1.0**: Higher creativity, more variation. Good for brainstorming, creative writing.

If you want the same output for the same input, use temperature 0. If you want variety, increase it.

**Prefilling (Assistant Turn Priming)**

This one surprised me. You can start the assistant's response to constrain its direction:

```python
messages = [
    {"role": "user", "content": "Analyze this code for issues"},
    {"role": "assistant", "content": "## Security Analysis\n\n1. "}  # Prefill
]
```

The model will continue from your prefill, forcing it into your desired format and framing. I use this for forcing specific output structures, skipping preambles, and setting the analytical frame.

### Technique 7: Chain-of-Thought Control

You can control whether and how the model shows its reasoning. I learned to match this to the context:

**When I Want Clean UX (User-Facing)**
```
Think through this step-by-step internally, but respond with ONLY:
1. The final answer
2. Your confidence percentage (0-100%)

Do not show your reasoning process.
```

**When I'm Debugging (See the Logic)**
```
Show your complete reasoning process in <thinking> tags.
Then provide your final answer in <answer> tags.
```

**When Stakes Are High (Traceable Steps)**
```
For each step in your analysis:
1. State what you are checking
2. Show the evidence
3. State your conclusion for this step

After all steps, provide final recommendation.
```

### Technique 8: The Meta-Prompt Pattern

This technique preemptively answers: "What should I do when X happens?" Without meta-prompts, the model falls back to defaults—which may not match your preferences.

```
<meta_instructions>
When my instructions are ambiguous:
- Ask ONE clarifying question (not multiple)
- If you must assume, state the assumption explicitly before proceeding
- Prefer the simpler interpretation

When you encounter an error or unexpected situation:
- Do not apologize extensively
- State what happened in one sentence
- Propose a specific next step

When I provide feedback:
- Incorporate it immediately without restating it
- Do not explain why your previous response was different
</meta_instructions>
```

### The Formula I Actually Use Now

Combining these techniques, here's what works:

```
Persona + Constraints + Output Format + Examples + Meta-Instructions = High Steerability
```

**Anti-patterns I've Hit Hard**

1. **Long prose paragraphs**: The model parses structure better than paragraphs. Use bullets, sections, and tags.
2. **Vague adjectives**: "Be thorough" means nothing. "Include at least 3 specific examples" is actionable.
3. **Implicit expectations**: If you want something, state it explicitly. The model cannot read your mind.
4. **Contradictory instructions**: "Be concise" and "explain thoroughly" conflict. I've done this more times than I'd like to admit.

## Part 2: When to Stop Using Agents

Here's the question that keeps me up at night: You've mastered prompt steerability. Your agents follow instructions precisely. So why would you ever need structured workflows?

The answer lies in understanding what agents fundamentally are—and what they're not.

### The Fundamental Truth About Agents

Even with perfect prompts, agents are **probabilistically correct**, not **deterministically correct**.

```
Structured Workflow:  Input → Step A → Step B → Step C → Output
                      (same path every time, predictable cost and latency)

Agent:                Input → LLM decides → maybe A → maybe B,C → maybe retry A
                      (variable path, variable cost, variable latency)
```

An agent might take 3 steps or 7 steps to accomplish the same task. It might succeed via an unexpected path. It might fail in ways you didn't anticipate. This variability is the source of both the agent's power (adaptability) and its weakness (unpredictability).

**The core question is not "can the agent do it?" but "should an agent do it?"**

### When I Learned Workflows Beat Agents

#### Lesson 1: When Correctness Outweighs Flexibility

Some domains have zero tolerance for error. I learned this when someone suggested using an agent for financial calculations. No. Just no.

**Medical Dosage Calculation**
```
# Workflow: Deterministic formula, auditable, verifiable
dosage_mg = patient_weight_kg * 0.5  # 0.5 mg/kg standard dose

# Agent: Could reason about the dose, but why introduce risk?
# An agent that is 99% correct means 1% of patients get wrong doses.
```

When the cost of a wrong answer exceeds the value of flexibility, use a workflow.

#### Lesson 2: When You Know the Decision Tree

If you can draw the logic as a flowchart, you probably don't need an agent. I burned $200 learning this.

```python
# Structured workflow - fast, cheap, predictable
def route_support_ticket(ticket):
    if "password" in ticket.lower():
        return "auth_team"
    elif "billing" in ticket.lower():
        return "finance_team"
    elif "bug" in ticket.lower():
        return "engineering"
    else:
        return "general"  # Only THIS case might benefit from an agent
```

**Cost Comparison I Measured**
- Workflow: ~1ms latency, negligible cost, deterministic
- Agent: ~500ms+ latency, $0.01-0.05 per call, variable behavior

When you know the decision rules, encoding them as code is faster, cheaper, and more reliable than asking an LLM to figure them out.

#### Lesson 3: When You Need Audit Trails

Compliance-heavy domains (finance, healthcare, legal, government) require explainable decision-making. I pitched an agent-based loan approval system once. Once.

```
Regulator: "Why did your system approve this loan?"

Agent answer: "The LLM determined based on various factors..."
              ❌ Not acceptable

Workflow answer: "Step 3 passed: income > 3x monthly payment.
                  Step 4 passed: credit score > 680.
                  Step 5 passed: debt-to-income ratio < 0.4.
                  See audit log entry #12847."
              ✅ Auditable and defensible
```

#### Lesson 4: When 95% of Cases Follow the Happy Path

Many systems have a predictable "happy path" that handles most inputs. Using an agent for the happy path is wasteful. I learned this processing vendor emails.

```python
def process_email(email):
    # Fast path: structured workflow handles 95% of cases
    if email.sender in KNOWN_VENDORS:
        return parse_vendor_template(email)  # Deterministic parser

    if email.has_standard_invoice_format():
        return extract_invoice_fields(email)  # Regex or ML classifier

    # Slow path: agent handles the 5% that need interpretation
    return agent.interpret_ambiguous_email(email)
```

**Architecture Principle**: Workflow first, agent as fallback. This optimizes for the common case while preserving flexibility for edge cases.

#### Lesson 5: When Latency Matters

Some systems cannot tolerate variable response times. Here's my decision table now:

| Use Case | Latency Budget | Agent Viable? |
|----------|---------------|---------------|
| Real-time bidding | 10ms | No |
| API rate limiting | <1ms | No |
| Fraud detection (real-time) | 50ms | Borderline |
| Chatbot response | 2-3s | Yes |
| Background processing | Minutes | Yes |

If your latency budget is under 100ms, agents are almost never appropriate.

### When Agents Actually Win

#### Victory 1: When Input Variability is High

When users can express the same intent in hundreds of different ways, agents excel. This is where I use them most.

```
User says: "Cancel my thing"
User says: "I want to stop the subscription I signed up for last month"
User says: "how do i unsubscribe?"
User says: "please discontinue my membership effective immediately"
User says: "nm dont charge me anymore"

Workflow approach: Would need hundreds of patterns, still miss variations
Agent approach: Understands intent naturally, handles novel phrasings
```

When you cannot enumerate all valid inputs, agents provide necessary flexibility.

#### Victory 2: When Context Changes Everything

Some decisions look simple but have context-dependent correct answers. I wrestled with refund policies for weeks before I realized this.

```python
def should_issue_refund(request):
    # This LOOKS like it could be a workflow, but...
    #
    # - Is this a first-time customer? (more lenient)
    # - Did they complain on social media? (reputation consideration)
    # - Is the item perishable? (no return possible)
    # - Did they use a promotional code? (different policy)
    # - Is it holiday season? (goodwill period)
    # - What is their lifetime value? (retention consideration)
    #
    # The decision tree explodes combinatorially.
    # An agent can weigh all factors naturally.
```

#### Victory 3: When Recovery Needs Judgment

Workflows fail rigidly. Agents can adapt. I saw this clearly when parsing API responses that kept changing format.

```
Workflow failure mode:
Step 3 failed → retry → retry → retry → FAIL → stop

Agent failure mode:
Step 3 failed → "Hmm, the API returned an unexpected format."
              → "Let me try parsing it differently."
              → "That worked. Continuing with Step 4."
              → Success via alternative path
```

If failures are common and creative recovery is valuable, agents provide resilience.

### The Pattern That Actually Works: Hybrids

In practice, the best architectures I've built use **structured workflows for orchestration** with **agents at specific decision points**.

```
┌─────────────────────────────────────────────────────────────────┐
│                STRUCTURED WORKFLOW (Orchestrator)                │
│                                                                  │
│  ┌────────────┐    ┌────────────┐    ┌────────────┐            │
│  │   Step 1   │───▶│   Step 2   │───▶│   Step 3   │            │
│  │ (workflow) │    │  (AGENT)   │    │ (workflow) │            │
│  └────────────┘    └────────────┘    └────────────┘            │
│       │                 │                  │                    │
│   Validate          Interpret          Execute                  │
└─────────────────────────────────────────────────────────────────┘
```

**Real Example: Invoice Processing**

```python
def process_invoice(pdf_file):
    # Step 1: WORKFLOW - Extract raw data (deterministic)
    raw_text = ocr_extract(pdf_file)

    # Step 2: WORKFLOW - Validate file format (deterministic)
    if not is_valid_document(raw_text):
        return reject("Invalid document format")

    # Step 3: AGENT - Interpret ambiguous fields
    structured_data = agent.extract_fields(
        raw_text,
        schema=INVOICE_SCHEMA,
        instructions="Extract vendor, amount, date, line items"
    )

    # Step 4: WORKFLOW - Validate against business rules
    validation_errors = validate_invoice(structured_data)
    if validation_errors:
        return reject(validation_errors)

    # Step 5: WORKFLOW - Store in database
    invoice_id = database.insert(structured_data)

    # Step 6: AGENT - Anomaly detection (judgment required)
    if structured_data.amount > REVIEW_THRESHOLD:
        review_decision = agent.should_flag_for_review(
            structured_data,
            context=get_vendor_history(structured_data.vendor)
        )

    return success(invoice_id)
```

**Why This Pattern Works**

1. **Predictable skeleton**: The overall flow is deterministic and auditable
2. **Intelligent joints**: Agents handle the parts that require interpretation
3. **Bounded uncertainty**: Agent decisions are constrained to specific, well-defined steps
4. **Graceful degradation**: If an agent fails, the workflow can route to human review

### My Decision Framework Now

Use this when designing a new system:

| Question | If YES | If NO |
|----------|--------|-------|
| Can I enumerate all valid inputs? | → Workflow | → Agent |
| Is there exactly ONE correct answer? | → Workflow | → Agent |
| Do I need sub-second latency? | → Workflow | → Agent |
| Is this subject to regulatory audit? | → Workflow | → Agent |
| Does context significantly change the answer? | → Agent | → Workflow |
| Do failures need creative recovery? | → Agent | → Workflow |

**The Mental Model I Use**

Think of agents as **expensive, intelligent consultants** and workflows as **cheap, reliable machines**.

You don't hire a consultant to do data entry for 10,000 rows—you use a script. You don't ask a machine to handle an angry customer who refuses to follow the phone tree—you escalate to a human (or agent).

**Architecture Principle**: Use deterministic workflows as the spine of your system, with agents positioned at specific decision points where judgment adds value.

## What I Know Now

The rise of LLM agents has created a temptation to solve every problem with intelligence. But intelligence has costs: latency, expense, unpredictability, and opacity. The art of building production AI systems lies in knowing when intelligence is worth those costs—and when simplicity wins.

**What I'd Tell Myself Six Months Ago**

1. **Steerability is achievable**: With structured prompts, few-shot examples, explicit constraints, and behavioral anchors, you can make LLM outputs highly predictable.

2. **Steerability is not determinism**: Even perfectly steerable agents are probabilistically correct. For some use cases, that's unacceptable.

3. **Workflows and agents are complementary**: The best architectures use workflows for structure and agents for judgment.

4. **Cost matters at scale**: An agent call that costs $0.01 becomes $10,000 at a million requests. I learned this watching our AWS bill.

5. **Start with workflows, add agents**: It's easier to add intelligence to a working system than to add reliability to an intelligent one.

The future of AI systems is not purely agentic or purely deterministic—it's hybrid architectures that use each approach where it excels. I'm still learning where those boundaries are, and I suspect they'll keep shifting as models get better and cheaper. But the framework holds: intelligence where it matters, reliability where it counts.

---

*What patterns have you found for balancing agents and workflows? I'd love to hear what's working in your systems.*
