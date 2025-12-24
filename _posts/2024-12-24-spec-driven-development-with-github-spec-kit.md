---
title: "GitHub Spec Kit and Spec-Based Programming: What I Learned Building with AI"
date: 2024-12-24 10:00:00 +0530
categories: [AI, Development]
tags: [github, spec-kit, ai-coding, claude-code, software-architecture, development-workflow]
toc: true
excerpt: "I spent months watching AI generate thousands of lines of code that worked in isolation but failed as a system. Then I discovered spec-driven development, and everything changed. Here's what I learned about flipping the development process upside down."
---

## The Problem That Kept Me Up at Night

You know that sinking feeling when you ask Claude Code to "build a task management app" and twenty minutes later you're staring at thousands of lines of code with no coherent architecture? I've been there. More times than I'd like to admit.

The authentication never quite matched my mental model. Data structures evolved organically during generation—which sounds nice until you realize "organically" means "chaotically." And when I asked for changes? The AI had no reference point for what the system *should* be. It was like giving directions to someone who forgot where we started.

This became my reality: **AIs excel at implementing details but struggle with maintaining architectural coherence across extended generation sessions.** They optimize locally—this function looks great, that component is perfect—without a global map. The result? Code that works beautifully in isolation but fails spectacularly as a system.

I burned through probably $200 in API costs learning this lesson the hard way.

## What Changed: Discovering Spec-Driven Development

Then I stumbled onto GitHub Spec Kit, and honestly, I was skeptical. Another tool promising to fix AI coding? But the core idea stuck with me: **flip the development process upside down.**

Instead of writing code and hoping it aligns with your vision, you write specifications first. Human-readable Markdown documents describing *what* the system should do. The AI uses these specs as a persistent reference, implementing code that stays true to your architectural intent across multiple sessions and refactorings.

What surprised me was how much this changed my workflow. I went from fighting with AI-generated code to collaborating with it. The difference? A shared blueprint we both understood.

## Understanding Spec-Driven Development

Here's what I wish someone had explained to me on day one: **Spec-driven development (SDD) is a methodology where specifications written in natural language serve as the authoritative source of truth for what a system should do.** Code is derived from specs, not the other way around.

### The Core Pattern I Now Follow

1. **Before writing code, I write a specification** describing the feature, API, or system behavior in structured Markdown
2. **AI agents read these specs** to understand requirements, constraints, and architectural decisions
3. **Code implements the spec**, and when specs change, code changes follow—maintaining alignment over time

### Why This Isn't Just Documentation

I learned this the hard way: traditional documentation is written *after* code exists and becomes stale within weeks. Specs in SDD are different:

- **Written first**: They exist before any implementation
- **Machine-readable**: Structured for AI consumption with specific sections and consistent formatting
- **Living artifacts**: Updated alongside code, serving as the single source of truth
- **Executable contracts**: AI agents validate code against specs continuously

Think of specs as "Markdown as a programming language." You're instructing the AI through prose rather than Python or JavaScript, but with the same level of rigor.

## Why This Actually Matters (The Problems I Solved)

### The Context Window Trap I Kept Hitting

Every AI coding session has a context window—the amount of information the model can "remember" at once. I kept running into this brutal trade-off:

- **Include too much code**: Context fills with implementation details, no room for strategic thinking
- **Include too little**: AI lacks architectural understanding and makes locally optimal but globally terrible decisions

Have you ever watched your carefully explained architecture disappear after 10,000 tokens? That was my Tuesday.

Specs solved this by **compressing architectural intent into human-readable prose.** I learned that a 2000-word specification can capture design decisions that would require reading 50,000 lines of code to infer.

**Real example from my bookstore project**: Instead of feeding Claude 10,000 lines of authentication code, I now provide a 500-word spec:

```markdown
## Authentication System

### Core Principle
Stateless JWT-based auth with refresh token rotation.

### Token Lifecycle
- Access tokens: 15-minute expiry, stored in memory only
- Refresh tokens: 7-day expiry, httpOnly cookies
- Rotation: New refresh token issued with each refresh

### Security Requirements
- bcrypt with cost factor 12 for password hashing
- CSRF protection via double-submit cookie pattern
- Rate limiting: 5 failed login attempts = 15-minute lockout
```

The AI now understands *why* tokens are structured this way. When I asked it to "add OAuth support," it preserved these constraints without me repeating them. That alone saved me hours.

### The Consistency Problem That Made Me Question Everything

I was building a feature over three days. On day one, I established naming conventions. On day three? The AI had "forgotten" those conventions because that conversation was buried 10,000 tokens ago. I spent half my time correcting stylistic drift.

This drove me nuts until I started using specs. Now they persist between sessions. The AI reads `constitution.md` (my project's core principles) and `specifications/` (individual feature specs) at the start of every session. Consistency by default.

### The Iteration Problem That Ate My Weekends

My old flow was exhausting:
1. Describe a feature to AI
2. AI generates code
3. I review and ask for changes
4. AI generates new code that contradicts earlier code
5. I spend the weekend reconciling conflicts

My spec-driven flow now:
1. I write a spec
2. I refine the spec with AI feedback
3. I approve the spec
4. AI implements
5. Changes flow from spec updates, not ad-hoc prompts

**Key insight I stumbled on**: It's WAY easier to iterate on a 1000-word prose document than on 5000 lines of generated code. You can edit, reorganize, and refine specs collaboratively before committing to implementation.

## The Four-Phase Workflow That Actually Works

Spec Kit structures the process into four phases. Each has dedicated commands. Here's how I use them:

### Phase 1: Specify (The Constitution)

**Command**: `spec init`

This creates a `constitution.md` file. Think of it as your project's founding document. I learned to include:

1. **Purpose**: What problem does this project solve?
2. **Core Principles**: Non-negotiable architectural decisions
3. **Constraints**: Technology choices, performance requirements, security policies
4. **Quality Standards**: Testing requirements, code style, review processes

**Example from my e-commerce bookstore**:

```markdown
# Bookstore Project Constitution

## Purpose
Build a lightweight, privacy-first online bookstore for independent publishers.

## Core Principles
1. **Privacy by Design**: No third-party tracking scripts. User data never leaves our servers.
2. **Performance**: Time to Interactive < 2 seconds on 3G connections.
3. **Accessibility**: WCAG 2.1 AA compliance, keyboard-navigable.
4. **Simplicity**: Prefer boring technology. PostgreSQL over NoSQL. Server-rendered HTML over SPAs.

## Technology Constraints
- Backend: Node.js + Express (team expertise)
- Database: PostgreSQL with row-level security
- Frontend: Progressive enhancement, vanilla JS + Hotwire Turbo
- Deployment: Single Docker container, horizontally scalable

## Quality Standards
- Test Coverage: Minimum 80% for business logic
- Code Review: Required for all changes
- Accessibility: Automated Axe tests on every PR
- Performance Budget: 200KB initial page weight
```

What clicked for me: when I ask Claude to "add a shopping cart," it now knows to:
- Avoid heavy JS frameworks (violates Simplicity principle)
- Implement server-side session storage (Privacy by Design)
- Add keyboard navigation (Accessibility standard)
- Include tests (Quality Standard)

All without me explicitly stating these in every prompt. This saved me from repeating myself 50 times.

### Phase 2: Plan (Breaking Down Features)

**Command**: `spec plan [feature-name]`

This creates `specifications/[feature-name].md`. Each spec follows a structured template that I've refined over time:

1. **Overview**: What does this feature do?
2. **Requirements**: Functional and non-functional
3. **Architecture**: Components, data flow, API contracts
4. **Implementation Notes**: Edge cases, error handling, performance considerations
5. **Testing Strategy**: What must be tested and how

**Real example from my shopping cart**:

```markdown
# Shopping Cart Specification

## Overview
Session-based shopping cart allowing users to add/remove books before checkout.

## Requirements

### Functional
- Add items with quantity selection (1-99)
- Remove individual items
- Update quantities
- Clear entire cart
- Persist cart across page reloads (same session)
- Display real-time total price

### Non-Functional
- Cart updates: < 100ms response time
- Support 1000 items per cart (edge case: bulk orders)
- Session timeout: 7 days of inactivity

## Architecture

### Data Model
```sql
CREATE TABLE cart_items (
  session_id UUID REFERENCES sessions(id),
  book_id INTEGER REFERENCES books(id),
  quantity INTEGER CHECK (quantity BETWEEN 1 AND 99),
  added_at TIMESTAMP DEFAULT NOW(),
  PRIMARY KEY (session_id, book_id)
);
```

### API Endpoints
- POST `/cart/items` - Add item (body: {book_id, quantity})
- PATCH `/cart/items/:book_id` - Update quantity
- DELETE `/cart/items/:book_id` - Remove item
- DELETE `/cart` - Clear cart
- GET `/cart` - Retrieve cart contents with book details
```

The patterns I've learned:

1. **Concrete Before Abstract**: Start with SQL schemas and API contracts (concrete) before discussing state management philosophy (abstract)
2. **Explicit Edge Cases**: Don't assume AI will infer "book removed from catalog" scenario—spell it out
3. **Constraints as Requirements**: "< 100ms response time" isn't a nice-to-have; it's testable
4. **Implementation Guidance**: "Denormalize book details" gives AI a specific optimization strategy

### Phase 3: Tasks (Making It Actionable)

**Command**: `spec task [spec-file]`

Spec Kit analyzes the specification and generates `tasks/[feature-name].md`. Each task is:
- **Atomic**: Completable in one focused session (1-3 hours for me)
- **Ordered**: Dependencies clear
- **Testable**: Success criteria explicit

**Why this matters for AI**: When implementing task #3 (POST endpoint), AI only needs the spec section on API design plus task description. It doesn't need to hold the entire cart system in context. This prevents context window overflow.

### Phase 4: Implement (Building with Guardrails)

**Commands**:
- `spec implement [task-file] [task-number]` - Implement specific task
- `spec verify [spec-file]` - Validate implementation matches spec

My implementation flow now:

1. **Load Context**: AI reads `constitution.md`, relevant spec file, task description
2. **Generate Code**: AI implements the task
3. **Generate Tests**: AI writes tests per the Testing Strategy section
4. **Verify**: Run `spec verify` to check spec alignment
5. **Mark Complete**: Check off task in task list

**Example session that actually worked**:

```bash
$ spec implement tasks/cart.md 3
📖 Reading context:
  - constitution.md
  - specifications/cart.md
  - tasks/cart.md (task 3: POST /cart/items endpoint)

🔨 Implementing...
  ✓ Created src/routes/cart.js
  ✓ Created tests/routes/cart.test.js
  ✓ Updated src/app.js (registered route)

✅ Implementation complete. Run tests:
  npm test tests/routes/cart.test.js

$ spec verify specifications/cart.md
🔍 Verifying implementation against spec...
  ✓ API endpoints match spec
  ✓ Database schema matches data model
  ✓ Error handling covers specified edge cases
  ✓ Response times within performance budget (avg 47ms)
  ⚠ Warning: Task 10 (removed books) not yet implemented

📊 Verification: 4/5 requirements met
```

The verify command became my safety net. It checks:
- Do API routes match the spec's endpoint list?
- Does the database schema include all specified columns/constraints?
- Are edge cases from "Implementation Notes" covered in code?
- Do tests exist for each item in "Testing Strategy"?

## Patterns I Discovered Through Trial and Error

### Pattern 1: Iterate on Specs Before Code

I learned to refine specifications collaboratively with AI *before* implementing. Here's a real example from adding search:

**My initial prompt**:
```
I want to add search functionality. Draft a specification for it.
```

**Claude's first draft was missing**:
1. Performance requirements (should be < 200ms with 1M books)
2. Fuzzy matching (users misspell author names constantly)
3. Filtering (search within genre)
4. Edge cases (empty query, special characters)

I asked it to expand, and we iterated **three times on the specification** before writing a single line of code. Each iteration added clarity, preventing implementation thrash.

**When I learned to stop refining**:
- I can visualize the implementation clearly
- Edge cases are explicitly handled
- Performance/security constraints are quantified
- Another developer (or AI) could implement it unambiguously

### Pattern 2: Specs as Refactoring Safety Nets

I needed to refactor the cart system to support guest checkout (previously required login). How do I ensure the refactor doesn't break existing behavior?

**My spec-driven refactor flow**:

1. **Update the specification first** with the changes
2. **Run `spec verify` before refactor** to establish baseline
3. **Implement refactor** following the migration strategy in the spec
4. **Run `spec verify` after refactor** to confirm no regressions
5. **Run existing tests** for extra safety

The spec served as a **regression test at the architectural level.** If verify passed after refactor, I hadn't accidentally broken core behaviors. This gave me confidence to make big changes.

### Pattern 3: Cross-Feature Coordination

When building Payment Processing and Order Confirmation in parallel, I learned to define contracts in both specs:

**specifications/payment.md**:
```markdown
## Output Contract
After successful payment, creates:
- Order record in `orders` table with status='confirmed'
- Emits event: `order.confirmed` with payload {order_id, user_id, total}
```

**specifications/order-confirmation.md**:
```markdown
## Input Contract
Listens for: `order.confirmed` event
Expected payload: {order_id, user_id, total}
```

Specs made **contracts explicit.** No more "these need to work together somehow"—we had precise event schemas both systems agreed on.

## What I Got Wrong (So You Don't Have To)

### Mistake 1: Over-Specifying Implementation

I initially wrote specs that dictated exact code structure. Big mistake. This removed all agency from the implementer. When a better pattern emerged, my spec became a hindrance.

**What I learned**: Specify *what* and *why*, not *how*. Let AI choose whether to use a class, function, or repository pattern—as long as it meets requirements.

### Mistake 2: Skipping the Constitution

I tried creating feature specs without a `constitution.md`. Every spec repeated "use bcrypt for passwords," "require 80% test coverage." Maintenance hell.

**The fix**: Establish a constitution first. Feature specs inherit these principles. Update once, applies everywhere.

### Mistake 3: Not Updating Specs During Implementation

I'd write a spec, implement it, discover edge cases, fix them in code... and never update the spec. The spec became fiction. Future AI read it and implemented the *old* behavior.

**Pattern I follow now**: When implementation reveals new edge cases:
1. Stop coding
2. Update spec with the edge case
3. Update tasks if needed
4. Resume coding

### Mistake 4: Treating Verify as Optional

I skipped `spec verify` sometimes, assuming alignment. Bad idea. I shipped 80% implementations—missing the 20% (rate limiting, error handling) that became production bugs.

**My rule now**: Make `spec verify` part of "done." Treat verify failures like test failures—blockers, not suggestions.

## When This Approach Shines (And When It Doesn't)

### Where I've Found Success

- **Greenfield projects**: Starting from scratch? Specs give you a blueprint
- **AI-heavy workflows**: The more I rely on AI code generation, the more critical specs become
- **Complex systems**: Microservices, event-driven architectures—specs document integration contracts
- **Long-lived projects**: Specs prevent architectural erosion over months/years

### Where I Still Struggle

- **Rapid prototyping**: If I'm spike-testing an idea in 2 hours, specs are overhead. I code-first, extract specs later if the prototype succeeds
- **Highly exploratory work**: When requirements change daily, lightweight bullet-point specs work better than rigid formal specs
- **Weekend hacks**: Solo projects where coordination doesn't matter—specs might be overkill

## What I'm Still Figuring Out

I'm not certain this approach is optimal for every team, but after trying three alternatives (code-first, documentation-after, and hybrid), spec-driven development is what stuck for me.

**Current limitations I'm navigating**:
1. **Manual spec writing**: Spec Kit doesn't auto-generate specs from code (yet)
2. **Verify accuracy**: Uses heuristics, not formal proofs. False positives happen
3. **Learning curve**: Teams resist "writing specs feels like documentation"

**What's working**: For projects that will live beyond 3 months, the upfront spec effort pays off dramatically around month 3 when refactoring becomes trivial instead of terrifying.

## If You're Going to Try This

Start small. Pick one new feature. Write the spec first. Let AI implement from the spec. Run verify. See if it clicks for you.

The moment it clicked for me: I asked Claude to add a feature six months after initial implementation. It read the spec, understood the architecture, and generated code that fit perfectly on the first try. No corrections. No "actually, we use JWT not sessions." It just *knew*.

That's when I stopped seeing specs as overhead and started seeing them as leverage.

## What's Next for Me

I'm experimenting with:
- Auto-generating specs from existing codebases (reverse engineering)
- Spec composition patterns for microservices
- Integration with design systems (UI specs linked to component libraries)

If you've found better ways to keep AI-generated code architecturally coherent, I'd genuinely love to hear about it. I'm still learning.

## Resources That Helped Me

**Getting Started**:
- [Spec Kit GitHub Repository](https://github.com/github/spec-kit) - Official toolkit
- [GitHub Blog: Spec-Driven Development with AI](https://github.blog/ai-and-ml/generative-ai/spec-driven-development-with-ai-get-started-with-a-new-open-source-toolkit/) - Comprehensive guide
- [LogRocket: Building with Spec Kit](https://blog.logrocket.com/github-spec-kit/) - Practical walkthrough

**Deeper Understanding**:
- [Martin Fowler: Understanding SDD](https://martinfowler.com/articles/exploring-gen-ai/sdd-3-tools.html) - Critical analysis
- [JetBrains: Spec-Driven Approach for AI Coding](https://blog.jetbrains.com/junie/2025/10/how-to-use-a-spec-driven-approach-for-coding-with-ai/) - Best practices

**Critical Perspectives** (these helped me see limitations):
- [Marmelab: The Waterfall Strikes Back](https://marmelab.com/blog/2025/11/12/spec-driven-development-waterfall-strikes-back.html) - Limitations for brownfield
- [Den Delimarsky: What's The Deal](https://den.dev/blog/github-spec-kit/) - Experimental nature

---

*Have you tried spec-driven development? What patterns have you discovered? I'm always looking to learn from others navigating AI-assisted development.*
