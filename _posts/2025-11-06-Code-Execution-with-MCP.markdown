---
title: "Code Execution with MCP - A Practical Guide"
date: 2025-11-06 10:00:00 +0530
categories: Technical
excerpt: "Learn how to use MCP tools with sandbox commands to save 95%+ of your token budget on data-heavy tasks."
---

I recently learned about a new pattern for using MCP tools with Claude. This post explains what it is and how you can use it today.

## What is MCP?

MCP stands for Model Context Protocol. It lets AI assistants like Claude connect to external tools and services. Think of it as a way for Claude to use your browser, fetch documentation, or access APIs.

The problem is that traditional MCP uses a lot of tokens. Tokens are like memory space for the AI. When you run out of tokens, the conversation gets expensive or stops working.

## The Token Problem

Here is what happens with traditional MCP. Let me give you a real example.

You ask Claude to find all the login buttons on a webpage. Claude takes a snapshot of the page. That snapshot has 15,000 lines of text describing every element on the page. All 15,000 lines get dumped into the conversation.

Claude reads through those 15,000 lines. It finds 3 buttons you care about. But those 15,000 lines stay in the conversation memory for the entire session. That uses about 30,000 tokens.

Now imagine you do this 5 times in one session. You have used 150,000 tokens just for raw data. Most of it is noise you never needed to see.

## The Code Execution Solution

Anthropic published a new pattern on November 4, 2025. The idea is simple. Instead of dumping all data into the conversation, Claude writes code to filter the data first.

Here is how it works. Claude takes the same webpage snapshot. But instead of reading all 15,000 lines, it writes a small script. The script saves the snapshot to a file. Then it uses grep to find only lines with "login" and "button". The script returns just 3 lines with the button IDs.

Only those 3 lines enter the conversation. That is about 500 tokens instead of 30,000 tokens. You save 98 percent of the tokens.

## Why This Works

The key insight is that large data never enters the conversation. Processing happens offstage in a sandbox. Only the filtered results come back to Claude.

Think of it like this. Traditional MCP is like someone reading you an entire phone book to find one name. Code execution is like them flipping through the phone book themselves and just telling you the page number.

## What You Need

Here is the important part. You do not need a special version of Claude. The current model already knows how to write code. The pattern uses existing capabilities.

What you do need is infrastructure to run the code safely. This is called a sandbox. Claude Code CLI already has sandbox support in beta. It is called the slash sandbox command.

## How to Use It Today

You can start using this pattern right now. You do not need to build anything custom. Just use the sandbox command manually.

Here is a step by step example.

### Example: Debug Website API Calls

Let me show you how to find failed API requests on a website. This is the kind of task that eats tokens.

**Step 1: Start with Sandbox**

Open Claude Code and type this:

```
/sandbox
```

This enables sandboxing. All bash commands now run in a restricted environment. Your MCP tools still work normally.

**Step 2: Capture Network Data**

Tell Claude:

```
Navigate to example.com and save all network requests to a file
```

Claude uses Chrome DevTools MCP to navigate. It captures all the network traffic. Then it saves the data to /tmp/requests.json.

The important part is that the raw data never enters your conversation. It goes straight to a file. That saves about 35,000 tokens right there.

**Step 3: Filter for Problems**

Now tell Claude:

```
Find all failed requests and show me just the URLs and error messages
```

Claude writes a command using jq. That is a tool for filtering JSON data. The command looks like this:

```bash
jq '.requests[] | select(.status >= 400) | {url, status, error}' /tmp/requests.json
```

The result is just 2 failed requests out of 47 total. That is about 400 tokens in your conversation instead of 35,000.

### Token Comparison

Let me show you the numbers:

**Traditional Approach:**
- Network data dumps into conversation: 35,000 tokens
- You find 2 failures
- Total: 35,000 tokens used

**Sandbox Approach:**
- Save data to file: 500 tokens (just confirmation)
- Filter with jq: 400 tokens (just results)
- Total: 900 tokens used

You save 97 percent of the tokens.

## Another Example: Browser Snapshots

Here is another common task. You want to find all submit buttons on a page.

**Traditional way:** Claude takes a snapshot. All 15,000 lines appear in the conversation. Claude scans through them. It finds 3 buttons. Total: 30,000 tokens.

**Sandbox way:** Claude takes a snapshot and saves to /tmp/page.txt. Then it runs this:

```bash
grep -i "button" /tmp/page.txt | grep -i "submit" | grep "uid-"
```

This finds the 3 buttons. Only the button IDs appear in the conversation. Total: 400 tokens.

That is 98 percent savings again.

## Real Working Session

You can try this right now. Here is a complete session you can copy:

```
$ claude
> /sandbox
> Navigate to httpstat.us in Chrome DevTools
> List network requests and save to /tmp/net.json
> Run: cat /tmp/net.json | jq '.[] | {url, status}'
> Take a screenshot and save to /tmp/page.png
```

This shows you how MCP tools and sandbox work together. The browser tools work normally. The data processing happens in the sandbox. Only summaries appear in your conversation.

## When to Use This

This pattern works best for tasks with large data:

- Network request debugging: 95 percent token savings
- Browser snapshot analysis: 96 percent savings
- Large documentation parsing: 85 percent savings
- Web scraping with filtering: 95 percent savings

It is not worth it for small tasks. If your data is under 5,000 tokens, just use normal MCP.

## Practical Tips

Here are some tips I learned:

**Use the filePath parameter.** Many MCP tools let you save directly to files. Chrome DevTools take_snapshot accepts a filePath argument. Use it.

**Chain Unix tools.** You can combine grep, awk, jq, and other tools in one command. This is powerful for filtering.

**Keep data between tasks.** Save your filtered data to organized directories like /tmp/debug/. You can compare results across multiple operations.

## Current Status

You might wonder if this pattern is available in Claude Code. The answer is not yet built in.

Anthropic published this as a pattern on November 4, 2025. They explained the concept but did not release code. Right now you need to use it manually with the sandbox command.

The good news is you get most of the benefits today. The manual approach gives you 80 to 90 percent of the efficiency gains. You just have to ask Claude explicitly to use files and filtering tools.

## What You Can Do

Here is my recommendation based on what I tested:

**Short term:** Use the sandbox command with MCP tools. Ask Claude to save large data to files. Use Unix tools to filter before bringing data into the conversation.

**Medium term:** Watch for updates. Either Anthropic will build this into Claude Code or the community will create implementations.

**Long term:** If you hit token limits frequently in 6 months and nothing official exists, consider building a custom implementation. But let others work out the bugs first.

## The Efficiency Formula

Here is a simple way to think about savings:

```
Savings = (Original Size - Filtered Size) / Original Size
```

For our network debugging example:
```
(35,000 - 500) / 35,000 = 98.6 percent
```

With a 200,000 token budget:
- Traditional: 5 to 6 network debugging tasks maximum
- Sandbox: 100 plus tasks possible

## Final Thoughts

This pattern is not revolutionary. It is just smart use of existing tools. The insight is simple. Keep large data out of the expensive conversation context. Process it in cheap scratch space. Only bring back the results you need.

You can use this today. Just type /sandbox at the start of your session. Then be explicit about saving data to files and filtering. You will see immediate token savings on data heavy tasks.

The best part is this works with your existing MCP tools. Chrome DevTools, Playwright, Hugging Face documentation, web fetching. They all benefit from this pattern.

Try it on your next debugging session. You might be surprised how much further your token budget goes.

---

**Update:** This post is based on Anthropic's November 4, 2025 article on code execution with MCP. The pattern is not yet built into Claude Code CLI. All examples use manual sandbox commands available today.
