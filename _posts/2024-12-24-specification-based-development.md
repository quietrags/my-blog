---
title: "Specification-Based Development: Why I Started Writing Docs Before Code"
date: 2024-12-24 10:00:00 +0530
categories: [Development, AI Agents]
tags: [ai-agents, claude-code, specifications, sdd, development-workflow, productivity]
toc: true
excerpt: "I spent $47 debugging a typo with an AI agent before I realized I was doing this all wrong. Here's what happened when I flipped my workflow and started writing specifications before code."
---

## The $47 Typo That Changed Everything

Last month, I ran a debugging session with Claude Code that cost me $47. The problem? A typo in a variable name. The AI agent spun through thousands of tokens trying different approaches, all because I hadn't been clear about what I actually wanted built.

That's when I discovered specification-based development (SDD), and honestly, it felt backwards at first. Write detailed documentation *before* coding? Isn't that the old Waterfall approach we all learned to avoid? But after three months of practice, I'm not going back. Here's why.

## What Is Specification-Based Development?

Think of SDD as flipping the traditional relationship between documentation and code. Instead of writing code first and documenting it later (if ever), you write detailed, version-controlled Markdown specifications that guide AI coding agents to generate production code.

Here's what surprised me: specifications aren't just advisory documents the AI might follow. They become executable instructions. You write "Users can filter search results by date range, category, and price" in a spec file, and the AI generates exactly that—including edge cases you might have forgotten to mention in a casual prompt.

## Why This Matters: The Unstated Assumptions Problem

Have you ever told an AI agent "add user authentication" and watched it make a dozen decisions you didn't mean it to make?

When I first tried this with Claude Code, the agent had to guess:
- Should passwords be hashed with bcrypt or Argon2?
- Do we need JWT tokens or session cookies?
- What's the password policy? 8 characters minimum? Special character requirements?
- Should registration send a confirmation email?
- What happens when users forget their password?

I was stuck in one of three bad patterns:
1. Specify everything in one massive prompt (which overloaded the context window)
2. Accept whatever the AI decided (hello, inconsistent architecture)
3. Iterate through dozens of follow-up prompts (and watch conversation memory degrade)

Before I found SDD, I spent 12+ hours writing PRDs, design docs, technical specs, and test plans for medium-sized features—only to have those documents become outdated the moment implementation began. Now? I write specifications once, the AI generates code from them, and the specs stay synchronized as living artifacts.

## How It Actually Works: Five Principles I Learned the Hard Way

### Principle 1: Specifications Are Executable, Not Wishful Thinking

My first instinct was to write vague requirements like "The system should handle authentication." That failed spectacularly. The AI generated something, sure, but it wasn't what I needed.

Then I tried this instead:

```markdown
## Authentication Implementation

**Endpoint**: POST /api/auth/login
**Required fields**: email (string, must match regex pattern), password (string, min 12 chars)
**Success response**: 200 with JWT token valid for 1 hour
**Error responses**:
- 400 if email format invalid
- 401 if credentials don't match
- 429 if more than 5 failed attempts in 15 minutes
```

The AI read this and generated code that implemented exactly these behaviors—including the rate limiting I'd have probably forgotten in a chat prompt.

### Principle 2: Separate "What" from "How" (or Risk Pain Later)

I learned this one when I switched from PostgreSQL to MongoDB six months into a project. My early specifications said things like "Store user data in PostgreSQL users table with columns: id, email, password_hash, created_at."

Wrong.

What worked better: "Users can be stored and retrieved by email address. System must support 10,000+ users with sub-100ms lookup times."

The "what" (requirements) stayed stable. Only the "how" (database choice) changed.

### Principle 3: Multi-Step Refinement Beats One-Shot Generation

My biggest early mistake was asking the AI to "build the shopping cart feature" in one go. It generated thousands of lines of code, and maybe 40% of it was actually usable.

What worked instead: breaking work into phases.

For that same cart feature, I now do:
1. **Specify**: "Users can add items, modify quantities, remove items, see total price"
2. **Plan**: Decides on data structures, API endpoints, state management
3. **Tasks**: Creates 8 separate tasks like "Implement addToCart endpoint," "Create CartItem model"
4. **Implement**: Generates code for each task individually

This prevents the "generate everything and hope" problem.

### Principle 4: Version Control Is Your Safety Net

I keep specifications in Git alongside code. Every change to requirements gets captured in commits, creating an audit trail.

Last Tuesday, a product manager requested "Allow guest checkout." Instead of a Slack message that would get lost, I updated the spec:

```diff
+ ## Guest Checkout Flow
+ Users can complete purchases without creating an account.
+ Collect: email (required), shipping address, payment details
+ After successful order, prompt: "Create account to track your order?"
```

That commit became the authoritative record of when guest checkout was approved, by whom, and what it entails. No more "wait, who decided this?" conversations.

### Principle 5: Let AI Agents Validate Themselves

Instead of writing "Build authentication" (which the AI will interpret however it wants), I now write tasks with clear acceptance criteria:

```markdown
Task: Create user registration endpoint
Acceptance criteria:
- Returns 201 on successful registration with user ID
- Returns 400 if email already exists
- Returns 400 if password < 12 characters
- Sends verification email to provided address
- Unit test coverage > 90%
```

The AI implements the endpoint, writes tests, and can verify whether its work meets these criteria before moving on. This dramatically reduced the "oops, the AI forgot about error handling" surprises.

## When This Actually Works (And When It Doesn't)

I'm not going to pretend SDD is perfect for everything. After three months, here's what I've learned:

### Where SDD Shines

**Greenfield projects**: Building from scratch with no existing code to navigate? Perfect fit. I built a bookstore application recently and 95% of the generated code compiled on first attempt.

**High-compliance environments**: I worked on a healthcare API that needed HIPAA compliance. Writing specifications created audit trails that mapped directly to regulatory requirements. The compliance team reviewed specs instead of code, cutting review time from 3 days to 4 hours.

**Team collaboration across tools**: When I switched from Copilot to Claude Code, the specifications stayed the same. That persistence prevented the usual "rewrite everything for the new tool" problem.

### Where SDD Falls Flat

**Exploratory work**: I tried using SDD for a novel fraud detection feature where we didn't know which ML algorithm would work. Writing detailed specs before understanding the problem was premature optimization at its worst. Use Jupyter notebooks and A/B testing for exploration; write specs *after* you've validated the approach.

**Small, trivial features**: Adding a "forgot password" link shouldn't require updating 4 specification files, 2 plan documents, and 6 task definitions. For 5-minute tasks, traditional AI-assisted coding is faster.

**Large, tangled codebases**: I hit a wall trying to use SDD on a 500K-line monolith. Context overload meant the AI agent couldn't understand the system well enough to implement specs correctly.

## The Real ROI: It's Not About Typing Less Code

Here's what surprised me most: the value of SDD isn't faster code generation. It's clearer problem understanding.

When I write "Users can add items to cart," I'm forced to think through:
- What if the item is out of stock?
- What's the maximum quantity per item?
- Does the cart persist across sessions?
- What happens to abandoned carts?

Traditional prompting ("build a shopping cart") lets me defer these questions until the AI generates code—then I iterate through follow-ups fixing gaps. SDD front-loads this thinking, reducing back-and-forth.

**Month 1-2**: My productivity actually *decreased*. Learning curve, specification writing overhead.
**Month 3-4**: Break-even. Specification time roughly equaled implementation time savings.
**Month 5-6**: Positive ROI. I'm saving 90+ minutes per feature from reduced documentation debt, fewer bugs, faster onboarding.

I almost quit at Month 2. Glad I didn't.

## The Tools I Actually Use

I tried three different SDD tools before settling on **GitHub Spec Kit**. Here's why:

**GitHub Spec Kit**:
- Open source (28K+ stars)
- Works with 18+ AI agents (Claude Code, Copilot, Cursor, Gemini)
- CLI-driven workflow: `/specify` → `/plan` → `/tasks` → `/implement`

**Kiro** was too opinionated for my taste (enforced structure felt restrictive), and **Tessl** is proprietary (vendor lock-in concerns).

**Pro tip**: Choose one tool and commit for 6 months before evaluating alternatives. I wasted a month switching between tools and rewriting specs in different formats. Switching costs are real.

## Patterns That Actually Work

After 50+ features built with SDD, here are the patterns I keep reusing:

### Pattern 1: Constitution + Feature Specs

Write project-wide standards once (tech stack, code conventions, compliance requirements) in a `constitution.md` file. Then write separate specs for each feature.

This prevents AI agents from making inconsistent choices. My Constitution says "All API endpoints follow REST conventions," so I never have to specify that in individual feature specs.

### Pattern 2: LessonsLearned.md Accumulation

I maintain a log of every mistake the AI makes:

```markdown
## 2024-12-05: Error Handling Gap

**Problem**: AI generated endpoints that returned 500 errors for validation failures
**Root cause**: Specification didn't define error response format
**Fix**: Added to Constitution: "All validation errors return 400 with RFC 7807 format"
**Impact**: 0 validation error bugs in 3 subsequent features
```

The AI reads this file as context for future work. Mistakes don't repeat.

### Pattern 3: Small, Testable Tasks

I learned this the hard way. "Implement user management" is too large and ambiguous.

What works: break into atomic units.
- Task 1: Create User model with validation
- Task 2: Implement registration endpoint (POST /users)
- Task 3: Implement login endpoint (POST /auth/login)
- Task 4: Write unit tests for User model validation

The AI can self-validate each task, reducing cascading errors.

## The Mindset Shift That Took Me Three Months

I've been coding for 15+ years. "Write code until it works" is burned into my brain. Specification-first felt unnatural at first.

But here's the shift that clicked for me:

**From: "Write code until it works"**
**To: "Define requirements clearly, let AI implement"**

**From: "Documentation is an afterthought"**
**To: "Specifications are the primary artifact"**

**From: "Iterate quickly, fix bugs later"**
**To: "Invest upfront in clarity, reduce bug rate"**

I'm still figuring this out. 60-80% usable generated code is what I consider a win (not 100%). AI agents make mistakes. The value is in reducing boilerplate and enforcing consistency, not eliminating code review.

## What I Got Wrong Initially

**Mistake 1: Specifying Everything**
I wrote 10,000-word specs that prescribed every implementation detail. The AI was locked into suboptimal choices with no room for intelligent decisions. Now I specify *what* (requirements) not *how* (algorithms) unless there's a performance reason.

**Mistake 2: Trusting "Testing Complete"**
An AI agent once claimed "all tests passing" but had actually documented manual testing procedures instead of writing unit tests. Always run tests yourself. Always.

**Mistake 3: Tool Hopping**
I switched from Spec Kit to Kiro to Tessl in the first month, never gaining proficiency in any. Pick one, commit for 6 months, then evaluate.

## Should You Try This?

I'm not certain this is optimal for everyone. But after trying three alternatives (traditional AI-assisted coding, pair programming with AI, and SDD), this is what stuck for me.

If you find something better, I'd love to hear about it.

**Try SDD if:**
- Your features have stable requirements (not changing daily)
- You're building greenfield projects or isolated features
- You value governance, audit trails, and long-term consistency
- You're willing to push through a 2-3 month learning curve

**Skip SDD if:**
- You're doing exploratory, research-driven work
- Your requirements change hourly
- You're working on trivial features (5-minute fixes)
- You can't tolerate upfront specification overhead

## What's Next for Me

I'm experimenting with specialized subagents (one for authentication, another for database operations, another for frontend) instead of one mega-agent with full project context. Context overload is real.

Also exploring RAG (Retrieval-Augmented Generation) to chunk large specifications. My current specs are hitting 10,000+ lines, which overloads context windows.

If you're trying SDD, what patterns are working for you? What am I missing?

---

**Further Reading**:
- [GitHub Spec Kit Repository](https://github.com/github/spec-kit) - Official documentation and installation
- [Martin Fowler: Understanding Spec-Driven-Development](https://martinfowler.com/articles/exploring-gen-ai/sdd-3-tools.html) - Critical analysis with limitations
- [Red Hat: How spec-driven development improves AI coding quality](https://developers.redhat.com/articles/2025/10/22/how-spec-driven-development-improves-ai-coding-quality) - Practical adoption guide
