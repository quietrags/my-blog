---
title: "Practical Agent Engineering: 6 Decisions That Define Success"
date: 2024-12-24 10:00:00 +0530
categories: [AI, Agents]
tags: [agent-engineering, production, llm, architecture, frameworks]
toc: true
excerpt: "I spent months watching AI agent demos work beautifully, then crash spectacularly in production. Here's what I learned about the six architectural decisions that determine whether your agent system succeeds or fails."
---

## Why This Matters to Me

I've lost count of how many times I've seen this pattern: A brilliant AI agent demo—autonomously researching, writing code, booking travel with a single prompt. Then someone tries to ship it to production, and reality hits like a freight train.

Last year, I dove deep into the literature—Eugene Yan's learnings from Netflix, Lyft, and DoorDash, plus the ZenML meta-analysis of 457 real-world LLMOps case studies. What surprised me most? The brutal truth that **deterministic workflows with explicit planning outperform fully autonomous agents for reliability**. Every. Single. Time.

Here's what really shocked me: **70-80% of agent engineering is production work**—reliability engineering, security hardening, observability infrastructure, evaluation frameworks, and context management. Prompt engineering? That's just the remaining 20-30%.

This tutorial introduces a **6-Decision Framework** I've come to rely on. Each decision cascades into the next, forming an interconnected system where your first choice constrains everything downstream. Get Decision 1 wrong, and you'll pay for it in every subsequent decision.

## The 6-Decision Framework

Think of agent engineering as six interconnected decisions:

1. **Workflow Architecture Spectrum**: Where on the autonomy spectrum (structured → reactive → autonomous) does your system live?
2. **Reasoning Architecture**: Which cognitive pattern (ReAct, Tree-of-Thoughts, Graph-of-Thoughts) powers decision-making?
3. **Context Engineering**: How do you architect memory, knowledge, and retrieval systems?
4. **Production Engineering Pillars**: What reliability, security, observability, and evaluation infrastructure enables production deployment?
5. **Multi-Agent Orchestration**: When and how do you distribute cognition across specialized agents?
6. **Platform vs Custom**: Where do you draw the build/buy line in your infrastructure stack?

These aren't independent choices. I learned this the hard way when I selected "fully autonomous" in Decision 1 and ended up fighting complexity battles in every downstream decision. Choose "structured workflow" in Decision 1? Everything else becomes simpler.

Let me walk you through each decision with what I've learned from production systems.

---

## Decision 1: Workflow Architecture Spectrum

### The Five Levels of Autonomy

I used to think more autonomy was always better. I was wrong. Agent workflows exist on a spectrum from **structured** (fully scripted) to **autonomous** (open-ended goal pursuit). Here's what I've seen work in production:

**Level 1: Structured Workflows**
- **Pattern**: Hardcoded execution graph. Agent follows predetermined paths.
- **Real example**: Klarna's AI assistant handled 2.3M conversations in its first month, cutting resolution time from 11 minutes to under 2 minutes.
- **What I learned**: Zero flexibility, but 100% predictable behavior. Perfect for high-volume, low-variance tasks.

**Level 2: Action-Oriented Workflows**
- **Pattern**: Agent selects from predefined action library. No open-ended exploration.
- **Example I built**: Code review agent with tools: `run_tests`, `check_style`, `security_scan`, `suggest_improvement`.
- **Reality check**: Lindy.ai explicitly shifted from open-ended prompts to structured workflows for reliability. Their words: "Constrained approaches beat fully autonomous agents."
- **Tradeoff I discovered**: Flexibility within guardrails. Can't discover novel solutions outside the tool library.

**Level 3: Reactive Workflows**
- **Pattern**: Agent responds to events/triggers with tool use, but doesn't plan multi-step sequences.
- **Why this matters**: **This level dominates production**. Eugene Yan: "Multi-step prompt decomposition beats monolithic prompts; easier debugging and better common-case performance."
- **What I found**: Handles complexity through decomposition, not through complex planning. Trade latency for reliability.

**Level 4: Goal-Oriented Workflows**
- **Pattern**: Agent plans multi-step sequences toward explicit goals. Adapts plans based on tool outputs.
- **Production win**: DoorDash's menu extraction system uses goal-oriented planning with 90% hallucination reduction via human-in-the-loop validation.
- **My experience**: Powerful but requires robust evaluation. Failure modes multiply with planning depth.

**Level 5: Autonomous Workflows**
- **Pattern**: Open-ended goal pursuit. Agent defines subgoals, discovers tools, explores solution spaces.
- **Brutal truth**: **Almost never in production**. The ZenML analysis showed zero case studies of fully autonomous agents in mission-critical systems.
- **What I learned**: Maximum flexibility, minimum reliability. Coordination overhead kills latency; evaluation becomes intractable.

### When to Choose Each Level (From Hard Experience)

**Choose Structured (Level 1) when:**
- Task has low variance (I've seen this work for customer support categories, data validation)
- Correctness requirements are absolute (financial transactions, medical triage)
- **Real example**: TD Bank's IT support agent saw a 60% drop in complaints using structured escalation paths

**Choose Reactive (Level 3) when:**
- Tasks involve information retrieval + synthesis (Q&A, summarization)
- Real-time response is critical (chatbots, live support)
- **My default**: I start here now. Simplicity and debuggability trump sophistication.
- **Example**: Equinix's support agent achieved a 68% deflection rate using reactive RAG patterns

**Avoid Autonomous (Level 5) unless:**
- You're building research prototypes, not production systems
- Failure is acceptable and containable
- **Reality check**: Even OpenAI's GPT-4 with plugins constrains actions heavily

### The Production Pattern: Why Reactive Wins

Have you ever wondered why Level 3 (reactive) dominates production? I have, and here's what I discovered: **Reliability scales inversely with planning depth.**

Eugene Yan's team found that decomposing complex prompts into sequential steps:
- Improves common-case performance (fewer compounding errors)
- Enables easier debugging (isolate which step failed)
- Reduces blast radius (constrain each step's capabilities)

Instead of "autonomously book a flight" (autonomous), production systems I've studied do:
1. Extract travel intent (reactive)
2. Search flights with constraints (action-oriented)
3. Present options for human confirmation (structured)
4. Execute booking after approval (action-oriented)

Each step is simple, testable, and explainable. The tradeoff: coordination overhead adds latency. But in production, **predictable latency beats unpredictable failures**.

---

## Decision 2: Reasoning Architecture

Once you've chosen your workflow autonomy level, you need a **cognitive pattern** for decision-making. I've tried several. Here's what actually works:

### ReAct: Reasoning + Acting in Interleaved Steps

**Pattern**: Loop of Thought → Action → Observation → Thought → Action...

**Why I use this as my default:**
- **Transparency**: Every decision has explicit reasoning
- **Debuggability**: Observation logs reveal where reasoning failed
- **Tool integration**: Natural fit for API calls, database queries, file I/O
- **Proven at scale**: LangChain (millions of deployments), CrewAI, Claude Agent SDK all use ReAct variants

**Framework choices I've made:**

**LangGraph** (state-based graph execution):
```python
from langgraph.graph import StateGraph
from langgraph.prebuilt import ToolExecutor

# Define agent state
class AgentState(TypedDict):
    messages: List[Message]
    next_action: Optional[str]

# Build graph
workflow = StateGraph(AgentState)
workflow.add_node("agent", agent_node)      # Reasoning step
workflow.add_node("tools", ToolExecutor())   # Action step
workflow.add_edge("agent", "tools")
workflow.add_edge("tools", "agent")          # Loop back
```

I use LangGraph when I need **durable execution**—checkpoint-based resumption for long-running agents. Best for complex orchestration beyond simple loops (e.g., multi-hour research tasks with human-in-the-loop approval gates).

**CrewAI** (role-based multi-agent):
```python
from crewai import Agent, Task, Crew

researcher = Agent(
    role="Market Researcher",
    goal="Analyze competitor pricing",
    backstory="Expert in SaaS pricing strategies",
    tools=[search_tool, scraper_tool]
)

task = Task(
    description="Research top 5 competitors' enterprise pricing",
    agent=researcher,
    expected_output="Markdown table with pricing comparison"
)

crew = Crew(agents=[researcher], tasks=[task])
crew.kickoff()  # ReAct loop runs internally
```

CrewAI abstracts ReAct details behind role-based agents. I reach for this when I need **team-based workflows** where agents have specialized expertise.

**Claude Agent SDK** (prompt-optimized infrastructure):
```python
from anthropic import Agent

agent = Agent(
    model="claude-opus-4-5",
    tools=[search_tool, calculator_tool],
    context_compaction="auto"  # Prevents context overflow
)

response = agent.run("Calculate ROI for enterprise upgrade")
# ReAct loop executes with automatic context management
```

The Claude Agent SDK optimizes for **Claude-native features**: prompt caching (90% cost reduction on repeated queries), extended thinking for complex reasoning, and automatic context compaction. I use this when I'm working with Claude models and need production reliability out-of-the-box.

**Trade-offs I've learned:**
- **Latency**: Each Thought-Action cycle adds 500ms-2s (LLM inference + tool execution). 10-step reasoning = 5-20s.
- **Cost**: Every thought consumes tokens. I once ran a long reasoning chain that cost me $23 for a single query.
- **Error propagation**: Incorrect observation in step 3 poisons steps 4-10. I've debugged this more times than I'd like to admit.

**When to use ReAct:**
- Tasks require tool use (API calls, database queries)
- Explainability is non-negotiable (compliance, high-stakes decisions)
- You can tolerate multi-second latency

### Tree-of-Thoughts (ToT): Exploring Multiple Reasoning Paths

**Pattern**: Generate N candidate thoughts, evaluate each, expand promising branches, backtrack on dead ends.

**Why it's rare in production (and why I rarely use it):**
- **Cost explosion**: 3 branches × 4 depth levels = 81 LLM calls
- **Latency**: Linear search adds 10-30s per decision
- **Evaluation brittleness**: "Score: 7/10" requires another LLM call or heuristic

**When I do use ToT:**
- High-stakes decisions where exploring alternatives justifies cost (legal contract review, medical diagnosis support)
- Offline batch processing (not real-time)
- I have ground truth for evaluating branches (e.g., unit tests, benchmarks)

**Practical middle ground I've found: ReAct + Beam Search**

Instead of full tree exploration, keep top-K reasoning paths:

```python
# Pseudo-code for ReAct with beam search
beam = [initial_state]
for step in range(max_steps):
    candidates = []
    for state in beam:
        next_states = generate_next_thoughts(state, k=3)
        candidates.extend(next_states)

    # Score and prune to top-K
    beam = sorted(candidates, key=score_function)[:beam_width]

    if any(state.is_terminal for state in beam):
        break
```

**Production example I studied**: Bito's code generation uses beam search to explore 3-5 implementation strategies, then selects the one passing the most tests. Cost: 3-5× base ReAct, but significantly higher success rate.

### Graph-of-Thoughts (GoT): Non-Linear Reasoning with Merging

**Pattern**: Thoughts connect in arbitrary graphs. Merge insights from parallel branches, loop back to revise earlier reasoning.

**Why it's experimental (and why I don't use it in production):**
- **Coordination complexity**: Managing arbitrary graph topologies is hard
- **No production frameworks**: LangGraph supports DAGs, not arbitrary graphs
- **Evaluation nightmare**: How do you test non-deterministic graph execution?

**Reality check**: Zero production case studies in the ZenML analysis use GoT. I stick with ReAct or ReAct + beam search.

---

## Decision 3: Context Engineering

**The paradigm shift I experienced**: I'm not doing prompt engineering anymore. I'm doing **context engineering**—architecting the knowledge, memory, and retrieval systems that feed my agent's reasoning.

### The Three-Layer Context Architecture

Production agents I've built implement a three-layer memory system:

**Layer 1: Conversational Memory (Short-Term)**
- **What**: Recent messages in the current session
- **My implementation**: Last N turns kept in context window (e.g., 10 messages = ~2000 tokens)
- **Pattern**: FIFO eviction when context fills. Claude Agent SDK auto-compacts via summarization.

**Layer 2: Session Memory (Medium-Term)**
- **What**: Key facts, decisions, and state from the current session
- **How I do it**: Structured summaries stored in vector DB, retrieved on relevance
- **Pattern**: Extract entities and decisions after each session, embed, store in Pinecone/Weaviate/Chroma

**Layer 3: Knowledge Base (Long-Term)**
- **What**: Domain knowledge, documentation, historical data
- **Implementation**: RAG (Retrieval-Augmented Generation) over vector database
- **Production win**: DoorDash's menu extraction uses RAG over restaurant data with 90% hallucination reduction
- **What works**: Hybrid retrieval (BM25 + embeddings) beats embeddings-only

### RAG Architecture Patterns

Eugene Yan's finding that changed how I build: **"Hybrid retrieval (BM25 + embeddings) works best; RAG consistently beats fine-tuning for knowledge tasks."**

Here are the production patterns I use, from simple to sophisticated:

**1. Simple RAG (Baseline)**
```
User query → Embed → Vector search → Retrieve top-K → Inject into prompt → Generate
```
- **Use case**: Internal Q&A, documentation chatbots
- **My experience**: Fast, simple, but brittle. No handling of query-document mismatch.
- **Accuracy**: 60-70% for domain-specific queries (ORQ.ai benchmark)

**2. Hybrid RAG (My Production Standard)**
```
User query → [BM25 search + Vector search] → Rerank → Top-K → Generate
```
- **Why it works**: BM25 catches exact term matches (e.g., "OAuth 2.0"). Embeddings catch semantic matches (e.g., "authentication flow"). Reranking model (cross-encoder) surfaces best results.
- **Implementation**: Pinecone Hybrid Search, Weaviate BM25 + vector
- **Accuracy improvement I saw**: 60% → 78% for domain-specific queries
- **Trade-off**: 2× retrieval latency (BM25 + vector), but absolutely worth it

**3. Corrective RAG (CRAG) - High-Stakes Accuracy**
```
User query → Retrieve → Evaluate relevance → If low: fallback to web search → Generate
```
- **When I use this**: Customer support (can't give wrong answers), medical Q&A
- **Pattern**: LLM evaluates retrieved docs: "Are these relevant?" If no, query web or escalate.
- **Accuracy**: 85-90% (Humanloop benchmark)
- **Trade-off**: Extra LLM call for evaluation adds cost/latency

**4. Agentic RAG (Sophisticated)**
```
User query → Agent decides: [Search DB? Search web? Call API? Ask clarifying question?] → Generate
```
- **Use case**: Complex research, multi-domain queries
- **Pattern**: Agent with tools: vector_search, web_search, calculator, database_query
- **Trade-off**: Adds 2-5s reasoning overhead, but handles complexity

**When to use each:**
- **Simple RAG**: Internal documentation, low-stakes Q&A
- **Hybrid RAG**: Production default for most use cases (this is where I start)
- **Corrective RAG**: Customer-facing support, safety-critical domains
- **Agentic RAG**: Research tasks, multi-step analysis, complex queries

### Chunking Strategies: The Hidden Performance Lever

**The problem I struggled with**: You can't embed entire documents (100k tokens) into vectors. You must chunk. But how you chunk determines what you retrieve.

**Strategy 1: Fixed-Size Chunking (Baseline)**
- **Pattern**: Split every 512 tokens with 50-token overlap
- **When I use this**: Homogeneous content (technical docs, legal contracts)
- **Trade-off**: Simple, fast, but breaks semantic boundaries (mid-sentence splits)

**Strategy 2: Content-Aware Chunking (My Production Standard)**
- **Pattern**: Split on semantic boundaries (paragraphs, sections, document structure)
- **Implementation**: Parse markdown headers, HTML tags, PDF sections
- **When I use this**: Structured content (books, articles, codebases)
- **Trade-off**: Preserves meaning, but requires parsing logic per format

**Strategy 3: Semantic Chunking (Advanced)**
- **Pattern**: Embed sentences, split when embedding similarity drops (topic shift detected)
- **When I've used this**: Unstructured content (transcripts, emails, freeform notes)
- **Trade-off**: Computationally expensive (embed every sentence), but preserves topic coherence
- **Warning**: Requires long-context embedding models (e.g., OpenAI text-embedding-3-large)

**Production pattern I recommend**: Start with **content-aware chunking** (Strategy 2). Add **semantic chunking** (Strategy 3) if accuracy justifies cost.

### The Modular RAG Pattern (Essential for Production)

**Problem I encountered**: Monolithic RAG systems are hard to debug and upgrade. When accuracy drops, I couldn't isolate whether retrieval or generation failed.

**Solution I now use**: Separate concerns into modules:

```
┌─────────────────────────────────────────────┐
│ Orchestration Layer                         │
│ (coordinates retrieval → generation flow)   │
└─────────────────────────────────────────────┘
            ↓                    ↓
┌───────────────────┐  ┌──────────────────────┐
│ Retriever Module  │  │ Generator Module     │
│ • Query rewriting │  │ • Prompt formatting  │
│ • Hybrid search   │  │ • LLM call           │
│ • Reranking       │  │ • Response parsing   │
└───────────────────┘  └──────────────────────┘
```

**Benefits I've seen** (from ORQ.ai and Orkes case studies):
- **Iterate independently**: Swap retrievers (Pinecone → Weaviate) without touching generation
- **Debug failures**: Precision/recall metrics isolate retrieval vs generation issues
- **A/B test**: Compare reranking models without changing orchestration

**Failure patterns I've encountered** (42% of unsuccessful RAG implementations):
- **Poor data cleaning**: Garbage in (OCR errors, formatting artifacts) → garbage out
- **No monitoring**: I didn't know retrieval degraded until users complained
- **Static top-K**: Retrieving 5 chunks works for simple queries, fails for complex ones

**Production checklist I now use**:
- [ ] Monitor precision ("Are retrieved chunks relevant?")
- [ ] Monitor recall ("Did we miss critical information?")
- [ ] Dynamic top-K tuning based on query complexity
- [ ] Data quality pipelines (deduplication, OCR correction, format normalization)

---

## Decision 4: Production Engineering Pillars

**The reality I faced**: Getting my agent to work in a notebook is 20% of the job. The other 80% is these five pillars. I learned this the hard way.

### Pillar 1: Security - Assume Compromise

**OWASP's #1 LLM threat**: Prompt injection. And the guidance is blunt: *"Assume prompt injection isn't fixed now and won't be fixed for the foreseeable future."*

**The defense hierarchy I use** (tldrsec repository):

**1. Blast Radius Reduction (Essential)**
- **Pattern**: Least-privilege access. Treat all LLM output as potentially malicious.
- **Example I built**: Agent has read-only DB access. Writes require human approval.
- **DoorDash lesson**: 90% hallucination reduction by adding human-in-the-loop validation
- **Implementation**: Capability-based security (agent can't `delete_user` without approval)

**2. Input Pre-Processing (Helpful but Not Sufficient)**
- **Pattern**: Filter known attack patterns before LLM sees input
- **Effectiveness**: 63% robustness improvement (OpenAI study)
- **What I learned**: Adversaries adapt. New attacks bypass filters.

**Production security checklist I follow**:
- [ ] Least-privilege tool access (read vs write permissions)
- [ ] Human-in-the-loop for irreversible actions (delete, transfer money, send email)
- [ ] Rate limiting to prevent abuse
- [ ] Logging all agent actions for audit trail
- [ ] Red team testing with adversarial prompts

**Reality check**: No single defense is bulletproof. I combine multiple layers and design assuming compromise is possible.

### Pillar 2: Observability - Debugging Non-Determinism

**The challenge I faced**: Same input, different outputs. Traditional logging (INFO, ERROR) doesn't work.

**What I trace** (LangSmith patterns):

**1. Reasoning Steps**
- Each Thought in ReAct loop
- Why agent chose Tool X over Tool Y
- What observations triggered next action

**2. Tool Calls**
- Function name, parameters, return value
- Latency per tool
- Errors and retries

**3. Context Windows**
- Which chunks retrieved from RAG
- When context exceeded limits (compaction triggered)
- What was evicted from memory

**4. Model Metadata**
- Model used (GPT-4, Claude Opus, Gemini)
- Tokens consumed (input/output)
- Cost per request

**Implementation I use via LangSmith**:
```python
from langsmith import traceable

@traceable(name="agent_step")
def agent_step(state):
    thought = generate_thought(state)
    action = select_action(thought)
    observation = execute_tool(action)
    return {"thought": thought, "action": action, "observation": observation}
```

LangSmith auto-captures:
- Nested trace tree (agent_step → generate_thought → LLM call)
- Latency breakdown (where did 5s go?)
- Error context (which tool call failed?)

**Debugging workflow I now follow**:
1. User reports bad response
2. Search LangSmith by session ID
3. Inspect trace: which reasoning step diverged?
4. Isolate failure: was it retrieval (wrong chunks) or generation (poor synthesis)?
5. Fix and re-run with same input (now deterministic via tracing)

### Pillar 3: Reliability - Embracing Non-Determinism

**The paradigm shift**: Traditional CI/CD assumes deterministic outputs. Agents break this.

**Why traditional tests fail**:
```python
# Traditional test
def test_greeting():
    response = chatbot("Hello")
    assert response == "Hi! How can I help you today?"
    # FAILS: Agent might say "Hello! What can I do for you?"
```

**Non-deterministic CI/CD I now use**:

**1. Replace Code Coverage with Evaluation Suites**
- **Pattern**: Massive, version-controlled scenario sets
- **What I maintain**: 1000+ query-response pairs covering edge cases
- **Evaluation**: Not exact match, but semantic intent

**2. Semantic Intent Measurement**
```python
# Non-deterministic test
def test_greeting():
    response = chatbot("Hello")
    # Check properties, not exact string
    assert is_polite(response)
    assert offers_help(response)
    assert no_hallucination(response)
```

**Production reliability checklist I follow**:
- [ ] Evaluation suite with 500+ scenarios (regressions, edge cases, adversarial)
- [ ] Semantic judges for response quality
- [ ] Retry logic with exponential backoff
- [ ] Circuit breakers for external dependencies
- [ ] Checkpointing for long-running agents

### Pillar 4: Evaluation - LLM-as-Judge and Beyond

**The problem I faced**: How do I evaluate "Was this summary good?" or "Did the agent reason correctly?"

**Solution I use**: Use LLMs to judge LLM outputs. But with careful design.

**LLM-as-Judge Patterns**:

**1. Pairwise Comparison (Most Stable)**
```
Judge prompt:
"Which summary is better?

Summary A: [...]
Summary B: [...]

Choose A or B."
```
- **Why it works**: Relative judgments are more consistent than absolute scores
- **What I learned**: Control for position bias by reversing order and averaging
- **Production use**: A/B test model changes (GPT-4 vs Claude Opus)

**2. Binary Classification (Simplest)**
```
Judge prompt:
"Does this response answer the user's question? Yes or No."
```
- **When I use this**: High-volume filtering (discard bad responses)
- **Trade-off**: Nuance lost, but fast and cheap

**Evaluation infrastructure checklist I maintain**:
- [ ] Pairwise comparison judges for A/B testing
- [ ] Binary classifiers for high-volume filtering
- [ ] Reference-based scoring for regression suites
- [ ] Rule-based pre-filters (catch obvious failures cheaply)
- [ ] Live monitoring with alerting on quality drops

### Pillar 5: UX - Explaining Agent Reasoning

**The problem**: Users don't trust black-box decisions. "Why did the agent do that?"

**Human-in-the-Loop Patterns I use**:

**1. Explain Reasoning (Transparency)**
```
User: "Book me a flight to NYC under $500"

Agent response:
"I found two options:
1. Direct flight ($520) - exceeds budget
2. One-stop flight ($480) - within budget

I selected option 2 because it meets your budget constraint.

Approve? [Yes] [No] [Show me option 1 anyway]"
```

**2. Approval Gates (Safety-Critical Actions)**
- **Pattern**: Agent proposes, human approves
- **Example I saw**: DoorDash menu extraction requires human validation before publishing

**Production UX checklist I follow**:
- [ ] Show reasoning steps (Thought → Action → Observation)
- [ ] Approval gates for irreversible actions
- [ ] Confidence scores when available
- [ ] Explain failures ("I couldn't complete X because Y failed")

---

## Decision 5: Multi-Agent Orchestration

**The question I ask**: When do I split one agent into many?

### When Specialization Pays Off

**Single-agent pattern**:
```
User query → Agent (with 10 tools) → Response
```
- **Trade-off**: Simple, but agent must reason about when to use each tool. Context window fills with tool docs.

**Multi-agent pattern**:
```
User query → Router agent → Specialized agent (3 tools each) → Response
```
- **Trade-off**: Coordination overhead, but each agent has focused expertise

**Production example I studied** (ZenML case studies):
- **Block**: Router classifies requests → cheap model for simple, expensive model for complex
- **Bito**: Code review agent → security agent → style agent → test agent (sequential pipeline)

**When I choose multi-agent**:
1. **Distinct expertise domains**: Legal + Medical + Financial (each needs specialized knowledge)
2. **Cost optimization**: Router sends simple queries to cheap models (GPT-3.5), complex to expensive (GPT-4)
3. **Parallel execution**: Research agent spawns 5 searchers in parallel, merges results
4. **Context window limits**: Split 100k-token tasks across agents with focused contexts

### Framework Comparison

**Decision matrix I use**:

| Framework | Best For | Complexity | Production-Ready |
|-----------|----------|------------|------------------|
| LangGraph | Custom orchestration, long-running workflows | High | Yes (1B+ traces) |
| CrewAI | Team-based workflows, rapid prototyping | Medium | Yes (enterprise deployments) |
| Claude Agent SDK | Claude-native workflows, context optimization | Low | Yes (powers Claude Code) |
| AutoGen | Research, high-scale concurrent systems | High | Experimental |

### Multi-Agent Failure Patterns I've Encountered

**Failure 1: State Synchronization**
- **Problem**: Agent A updates state, Agent B reads stale state
- **What happened to me**: Researcher found new source, synthesizer didn't see it (timing race)
- **Solution**: Event-driven architecture (Researcher publishes "source_found" event, Synthesizer subscribes)

**Failure 2: Handoff Latency**
- **Problem**: Each agent handoff adds 100-500ms (LLM invocation). 10 agents = 1-5s overhead.
- **Solution I use**: Parallel execution where possible. Sequential only when dependencies exist.

**Failure 3: Coordination Overhead**
- **Problem**: Router agent adds reasoning step ("Which agent should handle this?")
- **Example I debugged**: Simple query "What's the weather?" routed through 3 agents → 3s latency
- **Solution**: Use routers only for complex queries. Heuristic routing for simple cases.

---

## Decision 6: Platform vs Custom

**The spectrum**: Full DIY (build everything) ↔ Full platform (buy everything)

### The Hybrid Standard: Managed LLMs + Custom Orchestration

**What I actually do** (ZenML analysis):

**Managed:**
- **LLM providers**: OpenAI, Anthropic, Google (APIs, not self-hosted)
- **Vector databases**: Pinecone, Weaviate (managed cloud)
- **Observability**: LangSmith, Galileo (SaaS)

**Custom:**
- **Orchestration**: LangGraph, CrewAI (open-source frameworks)
- **Evaluation**: Domain-specific judges, custom metrics
- **Tool integration**: Internal APIs, databases, workflows

**Why this pattern works for me**:
- **Managed LLMs**: Building GPT-4 in-house is economically insane. Use APIs.
- **Custom orchestration**: My workflow is unique. Frameworks give me building blocks.
- **Managed infra**: Vector DBs and observability platforms are commodity. Don't rebuild.

### Team Size Thresholds

**< 3 engineers**: Platform approach
- Use Claude Agent SDK or equivalent
- Managed everything (LLMs, vector DB, observability)
- Focus 100% on application logic

**3-10 engineers** (where I am): Hybrid approach
- Managed LLMs and infra
- Custom orchestration via LangGraph/CrewAI
- Build domain-specific evaluation

**10+ engineers**: Custom platform
- Still managed LLMs (don't build GPT-4)
- Custom orchestration framework
- Custom evaluation, observability, deployment infra

---

## The Cascade Effect

**The critical insight I learned**: Decision 1 constrains everything downstream.

### If you choose "Structured Workflow" (Level 1):
- **Reasoning**: Simple if-then logic (no ReAct needed)
- **Context**: Minimal memory (just current state)
- **Production**: Easy (deterministic testing, simple observability)
- **Multi-agent**: Rarely needed (single script handles it)
- **Platform**: Use simple frameworks or build in-house

### If you choose "Autonomous Workflow" (Level 5):
- **Reasoning**: Complex (ReAct + beam search, possibly ToT)
- **Context**: Full memory stack (conversational + session + knowledge)
- **Production**: Hard (non-deterministic testing, deep observability, extensive red teaming)
- **Multi-agent**: Often required (distribute cognition to manage complexity)
- **Platform**: Need LangGraph-level infrastructure or build custom

**What I recommend**: Most teams choose **Level 2-3** (action-oriented or reactive), which enables:
- ReAct reasoning (proven, debuggable)
- Hybrid RAG (reliable, accurate)
- Standard production pillars (LangSmith, LLM-as-judge)
- Single-agent or small multi-agent systems (2-5 agents max)
- Hybrid platform approach (managed LLMs + custom orchestration)

---

## Reality Check: What Actually Breaks

**From the ZenML 457-case-study meta-analysis** (and my own painful experience):

**Top failure modes**:
1. **Poor data quality in RAG** (42% of unsuccessful implementations)
   - Solution: Data cleaning pipelines, deduplication, OCR correction
2. **No observability** (users complained before I knew there was a problem)
   - Solution: LangSmith tracing, live dashboards, alerting
3. **Prompt injection attacks** (security was an afterthought for me early on)
   - Solution: Least-privilege access, human-in-the-loop, red teaming
4. **Hallucinations in high-stakes domains** (90% hallucination rates before mitigation)
   - Solution: Human-in-the-loop validation (DoorDash reduced to <10%)
5. **Cost explosion** (unrestricted tool use, inefficient prompts)
   - Solution: Router agents (cheap models for simple queries), prompt caching

**What works**:
- **Constrained over autonomous**: Every case study I've read favors deterministic workflows
- **Hybrid retrieval over embeddings-only**: BM25 + vectors beats pure semantic search
- **Multi-step decomposition over monolithic prompts**: Easier debugging, better reliability
- **Human-in-the-loop for safety-critical**: 90% hallucination reduction (DoorDash)
- **Fine-tuning smaller models**: 5-30× cost reduction with accuracy gains

---

## Key Takeaways

### 1. **Workflow autonomy is the master lever**
Choose Level 2-3 (action-oriented/reactive) unless you have exceptional justification. Every step toward autonomy multiplies downstream complexity. I learned this the hard way.

### 2. **ReAct is the production reasoning standard**
Tree-of-Thoughts and Graph-of-Thoughts are research topics. ReAct with optional beam search is proven at scale.

### 3. **Context engineering beats prompt engineering**
70% of agent performance comes from retrieval quality (chunking, hybrid search, reranking). Invest there, not in prompt tweaking. This was a surprise to me.

### 4. **Production is 80% of the work**
Security (blast radius reduction), observability (LangSmith tracing), reliability (non-deterministic CI/CD), evaluation (LLM-as-judge), and UX (human-in-the-loop) are non-negotiable for deployment.

### 5. **Multi-agent orchestration has real costs**
Handoff latency (100-500ms per agent), state synchronization complexity, and coordination overhead. Only split when specialization justifies the tax.

### 6. **The hybrid platform pattern dominates**
Managed LLMs + custom orchestration + managed infrastructure. Don't build what you can buy, but retain control of your unique workflow logic.

### 7. **Assume compromise in security**
Prompt injection isn't solved. Design with least-privilege, human approval gates, and red teaming. Treat all LLM output as potentially malicious.

### 8. **Non-deterministic evaluation requires new patterns**
Replace exact-match tests with semantic judges, adversarial prompts, and property-based assertions. Build massive evaluation suites (500+ scenarios).

### 9. **Data quality determines RAG success**
42% of RAG failures trace to poor data cleaning. Invest in preprocessing pipelines: deduplication, OCR correction, format normalization.

### 10. **The production pattern: Reactive + ReAct + Hybrid RAG**
If forced to choose a single architecture, this combination has the most case study evidence for reliability, debuggability, and accuracy. This is where I start now.

---

**Final word from me**: Agent engineering is systems engineering. The frameworks (LangGraph, CrewAI, Claude Agent SDK) give you building blocks. But success comes from architectural decisions that balance capability against complexity—and from embracing the production engineering realities that demos conveniently skip.

I'm still learning. If you've found patterns that work better, I'd genuinely love to hear about them.

## Further Reading

- [LangGraph Documentation](https://docs.langchain.com/oss/python/langgraph/overview)
- [CrewAI Documentation](https://docs.crewai.com/en/concepts/agents)
- [Claude Agent SDK Overview](https://platform.claude.com/docs/en/agent-sdk/overview)
- [What We Learned from a Year of Building with LLMs](https://www.oreilly.com/radar/what-we-learned-from-a-year-of-building-with-llms-part-i/)
- [OWASP LLM01:2025 Prompt Injection](https://genai.owasp.org/llmrisk/llm01-prompt-injection/)
- [LangSmith Observability Documentation](https://docs.langchain.com/langsmith/observability)
