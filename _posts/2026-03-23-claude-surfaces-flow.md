---
layout: single
title: "Twelve Surfaces, One Model, and the Pathways Nobody Drew"
date: 2026-03-23
categories: [AI]
tags: [claude, anthropic, developer-tools, workflows, product-design]
toc: false
excerpt: "Claude's product surface area keeps expanding. The official framing is 'same model, many surfaces.' What's missing is the map of how you actually move between them."
---

I have been using Claude Code as my default for nearly everything. Chat, brainstorming, building artifacts, managing projects, all of it routed through the terminal. When you spend enough time in one tool, it becomes the lens through which you see every task. Need a diagram? Prompt Claude Code to generate SVG. Need to organize research? Create a directory and dump specs into it. Need a quick answer? Start a conversation right there in the CLI.

It works. Mostly. But there is a growing feeling that I am forcing a single surface to do the work of several, and that the friction I keep running into is not a limitation of the model but a mismatch between the surface and the task.

Anthropic now offers something like twelve distinct surfaces through which Claude operates. Desktop app, web chat, projects, artifacts, code execution, Cowork, Claude Code, Chrome extension, Research, API, MCP connectors, Excel and PowerPoint add-ins. The official framing is clean: one model, many surfaces, differentiated by what context Claude can see, what actions it can take, and how much autonomy it has. I spent time mapping all of this out recently, trying to build a mental model I could actually use.

The taxonomy helped. But what I found more interesting was something the taxonomy does not capture.

## The blank paper problem

Here is a concrete example of surface mismatch. I need a diagram, say an auth flow showing how a user request moves through token validation, session management, and role-based access. In Claude Code, this means starting from nothing. I have to describe the format I want (HTML? SVG? Mermaid?), specify dimensions, explain the layout, and then iterate through rounds of "no, move that box to the left" and "the text is overflowing." There is no default visual vocabulary. You are staring at a blank page every time.

The Claude desktop app recently added native diagramming. You describe the same auth flow and it draws one. No format negotiation, no HTML scaffolding. The difference in friction is real.

![The same diagramming task in Claude Code versus Claude Desktop](/my-blog/assets/images/posts/claude-surfaces-flow/blank-paper.svg)

I had been absorbing that friction in Claude Code without questioning it, assuming it was just how things worked. The desktop's diagramming capability made me realize I was using the wrong surface for visual work. Not because Claude Code cannot do it, but because it was never designed for it. Asking Claude Code for a quick diagram is like writing a letter in a spreadsheet. The tool can do it, but every step fights you.

This is not a complaint about Claude Code. It is genuinely good at what it does. But the moment I need something visual and conversational rather than code-based, I am on the wrong surface. I just did not have a framework for knowing that.

## Where your inputs live

The most useful idea I encountered while mapping this out is simple: pick the surface based on where your inputs are and where your outputs need to land.

![Input types mapped to Claude surfaces](/my-blog/assets/images/posts/claude-surfaces-flow/input-routing.svg)

If the inputs are in your head, half-formed ideas, questions you are still figuring out how to ask, then chat is the right surface. I think of chat as the tool for when something is in your head and you want to get it somewhere. Last week I wanted to understand the tradeoffs between building a custom agent framework versus using an off-the-shelf one. I did not have a spec or a document. I had a feeling that something was off about the approach we were taking, and I needed to talk it through. That is a chat. The model thinks with you and helps you figure out what you are actually asking.

But chat on its own has a problem: context dies when the conversation ends. I had a series of conversations about Claude's product surfaces that built on each other over several days. Each time I started a new session, I had to re-establish the framing, re-explain what I had figured out, and get back to where I was. The context I had accumulated was gone.

If the inputs are documents, PDFs, specs, prior work, reference material, then Projects give you a persistent workspace where every conversation inherits that context. Think of it as setting up a desk. You arrange your references once and then have many conversations at that desk. For something like a client engagement where you keep returning to the same account history, competitive landscape, and strategic context, a project means you never start from zero.

If the inputs are files on your local machine and the work is transformative, reorganizing, synthesizing, producing new documents from existing ones, that is Cowork territory. You have fifty meeting notes from the last quarter and you want a synthesized project brief. The starting material is already on disk.

If the inputs are code, Claude Code is the right place. You want to refactor an auth module, add test coverage, debug a failing pipeline. The starting material is a repository.

This routing heuristic covers maybe eighty percent of decisions. But it also raises a question I have not fully resolved.

## Cowork and Claude Code feel interchangeable

On paper, the distinction is clear. Claude Code is code-first. Cowork is document-first.

![Cowork versus Claude Code comparison](/my-blog/assets/images/posts/claude-surfaces-flow/cowork-vs-code.svg)

Claude Code operates in a repository context with git, tests, and build systems. Cowork operates on files and folders without assuming a development workflow. In theory, you pick Cowork when the task is "organize these expense receipts and build a spreadsheet" and Claude Code when the task is "refactor the payment module and add integration tests."

In practice, I cannot think of anything I can do with Cowork that I cannot also do with Claude Code. Claude Code reads files, writes files, reorganizes directories, produces documents. I have used it to synthesize research notes, organize specs, and generate project briefs, none of which involve code. The boundary feels more like a suggestion than a wall.

I suspect the distinction matters more when you are doing pure knowledge work with no code involved, where Cowork probably offers a smoother experience because it does not carry the overhead of a development-oriented mental model. If you are a project manager organizing deliverables, or a consultant synthesizing interview transcripts, Cowork is probably the more natural fit. But I have not pushed hard enough on Cowork to say that with confidence.

This is one of those areas where the twelve-surface taxonomy looks clean on a slide but feels muddy when you actually sit down to work.

## The progression nobody drew

What I find more interesting than the taxonomy is the flow between surfaces. There is a natural progression that follows how thinking actually works, and I did not see it until I tried to articulate why I keep defaulting to Claude Code for everything.

You start in chat. Something is in your head, vague, unformed, maybe just a question you cannot articulate cleanly yet. You talk it through. I spent three separate sessions in Claude exploring what Anthropic's product surface area actually looks like. Each conversation built on the previous one, each one refining the mental model further. Over those conversations I accumulated context: the three-axis framing (context, action, autonomy), the input-output routing heuristic, a clearer picture of where each surface fits.

At some point you do not want to re-derive all of that context every time you start a new conversation. You want a project, a persistent workspace that holds what you have learned so that every new conversation starts where the last one left off. The chat-to-project transition is the moment when exploration becomes a workstream.

Once you have a project with accumulated knowledge, the next natural step is making something with it. A visualization. An interactive prototype. A document you can share. That is what artifacts are for. They are the output surface, where accumulated understanding turns into something you can see and iterate on. You have spent a week building up a framework in conversations and project context, and now you want to see it as an interactive diagram or a polished document.

![Two pathways through Claude's surfaces](/my-blog/assets/images/posts/claude-surfaces-flow/progression-flow.svg)

Chat to Projects to Artifacts. Vague idea to accumulated context to tangible output. This is not a taxonomy. It is a pathway, and it mirrors the arc of how knowledge work actually progresses.

Cowork and Claude Code are not steps on this same ladder. They are separate entry points. You reach for Cowork when the starting material is already on disk in the form of files. You reach for Claude Code when the starting material is a codebase. They join the flow at a different point, where you already have something concrete and want to transform it.

## The feature I keep wishing existed

If the progression from chat to project is natural, it should also be easy. Right now it is not.

I have long conversations in Claude Code where I explore a topic, build up a mental model, arrive at frameworks and decisions. The conversation about Claude's surfaces is a good example. Over three sessions I developed the input-output routing idea, figured out the difference between the ideation pathway and the material-on-disk pathway, and identified where the product gaps are. All of that context lives in those conversations. When I wanted to turn it into something persistent, a knowledge base I could reference in future sessions, I had to do it manually. Create a directory, write specs, organize notes. The synthesis step was entirely on me.

What I wish existed is a way to take a rich conversation and synthesize it into a project.

![Conversation synthesis into a structured project](/my-blog/assets/images/posts/claude-surfaces-flow/synthesis-wish.svg)

Not a transcript dump. An actual synthesis: here are the key ideas organized by theme, here are the instructions Claude figured out for how to work with you on this topic, here is the knowledge base so you do not have to ask the same questions again.

In Claude Code, this would mean being able to say "take the last three sessions and turn them into a CLAUDE.md and a set of reference documents." On Claude.ai, it would mean a chat-to-project pipeline that does the organizing for you. Something like a button that takes the sprawl of a long conversation and distills it into a workspace.

I currently do this by hand. In Claude Code, I write specs and CLAUDE.md files after the fact, trying to capture what I learned. In Claude.ai, I start a project and manually paste the important bits from previous chats. Both approaches lose something in translation. The conversational flow where ideas were refined through back-and-forth gets flattened into static documents.

This does not exist today.

## The island problem

There is an irony in Anthropic's product architecture that is worth naming, and I say this as someone who uses these tools every day and genuinely likes them.

The official framing is "same model, many surfaces." The surfaces are supposed to be different windows into the same intelligence. But the surfaces are islands.

![Claude's surfaces as disconnected islands](/my-blog/assets/images/posts/claude-surfaces-flow/island-problem.svg)

A conversation in Claude.ai cannot be accessed from Claude Code. A project on Claude.ai has no connection to a directory in Cowork. Memory exists, but it is a thin layer of synthesized facts, not the rich context from the conversations that produced those facts. There is no API for accessing your own chat history. No programmatic bridge between surfaces.

Consider a practical scenario. I have a project on Claude.ai where I have uploaded client briefs, competitive analysis, and strategic context for an engagement. Now I want to use Claude Code to build a prototype that draws on that same context. I cannot. I have to re-establish the context in Claude Code, either by copying documents into a local directory or by re-explaining the background in conversation. The two surfaces, both running the same model, both operating on the same body of knowledge, cannot see each other.

I am not even sure the model is truly identical across them. The source material I was working from presents "same model" as a foundational truth, but I have noticed what feel like differences in how Claude responds in Claude Code versus Claude.ai. I have no hard evidence. It could be the system prompts, the tool configurations, the context windows. But "same model" is something I am taking on faith rather than something I can verify.

The practical result is that I use Claude Code for everything not because it is the best surface for every task, but because switching surfaces means losing context. The cost of picking the right tool is starting over. An AI company building tools for knowledge work has not yet solved knowledge continuity across its own products. The individual surfaces are good. Some of them are very good. What is missing is the wiring between them.

## What I actually wanted

When I set out to map Anthropic's surface area, I was looking for a taxonomy. A clean chart: use this surface for that task. What I found instead was a set of pathways.

If the idea is in your head, start in chat. When you have accumulated enough context to want persistence, move to a project. When you are ready to create, use artifacts. If your starting material is files, start with Cowork. If it is code, start with Claude Code.

The taxonomy is useful for orientation. But the daily question is not "which of twelve surfaces do I pick?" It is "where am I in the arc of this work, and what surface fits this stage?"

I think the pathways between surfaces should be as well-designed as the surfaces themselves. That is the part that is not there yet.

---

*This post was prompted by a conversation exploring Anthropic's product landscape as of March 2026. The three-axis framing (context access, action space, autonomy level) and the input-output routing heuristic both come from that exploration. The pathways and the gaps are my own observations from daily use.*
