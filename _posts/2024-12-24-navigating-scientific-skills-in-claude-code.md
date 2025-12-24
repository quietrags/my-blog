---
title: "Navigating the Scientific Skills Toolkit in Claude Code"
date: 2024-12-24 10:00:00 +0530
categories: [AI, Development]
tags: [claude-code, scientific-computing, bioinformatics, productivity, workflows]
excerpt: "I spent weeks exploring 169 scientific skills in Claude Code, trying to figure out when to use each one. Here's what I learned about making the right choice between skills and Context7, and how to craft prompts that actually work."
toc: true
---

## The Problem I Kept Running Into

Have you ever opened Claude Code, seen a list of 169 scientific skills, and thought "which one do I actually need?" That was me every single day for the past month. I'd waste 15 minutes just scrolling through skill names, trying to figure out if I needed `scikit-learn` or `statistical-analysis` for my classification problem.

The worst part? Half the time I'd choose wrong. I'd invoke a skill expecting code, get a conceptual explanation instead, then have to start over. Or I'd use a heavy 13k-token skill when a quick Context7 lookup would have been enough.

This guide exists because I finally sat down and mapped out the entire landscape. If you work with scientific computing in Claude Code, this might save you the trial-and-error I went through.

## Skills vs Context7: The Decision I Always Got Wrong

Here's what took me embarrassingly long to understand: **Context7 is for syntax, Skills are for strategy.**

I burned through tokens for weeks before this clicked. Let me give you a real example that happened last Tuesday:

I was debugging a BioPython error: `ValueError: Invalid sequence character`. My first instinct? Invoke the `biopython` skill (3.4k tokens). Wrong move.

What I actually needed was Context7 to look up the function signature and see which characters were valid. The skill would have given me a 2000-word essay on sequence manipulation best practices when I just needed one line from the docs.

### When I Use Context7 Now

- "What parameters does `pandas.read_csv()` accept?"
- "Why is `sklearn.model_selection.cross_val_score` throwing this error?"
- "What's the syntax for Scanpy's normalization function?"

### When I Use Skills

- "How do experts handle imbalanced classification with 90/10 splits?"
- "What's the standard single-cell RNA-seq pipeline?"
- "Should I use Random Forest or XGBoost for this dataset?"

The distinction sounds obvious now, but it wasn't when I started. The rule that finally stuck: **if your question starts with "what parameters" or "why is this function," use Context7. If it starts with "how do experts" or "what's the right approach," use a skill.**

## The Two Modes I Wish Someone Had Explained Earlier

This was my second big mistake. I kept treating skills like they only existed to generate code. Turns out, there are two completely different ways to use them:

### Question & Answer Mode: The Planning Phase

This is where you're still figuring things out. You haven't committed to an approach yet. You want expertise, not implementation.

Here's a prompt I actually used last week:

```
/scientific-skills:scikit-learn

I have 10k samples, 50 features, imbalanced classes (90/10 split).
Don't write code yet.

Which algorithms should I consider and why?
What preprocessing would you recommend?
```

Notice that **"Don't write code yet"**? That phrase saved me from getting 200 lines of implementation when I just wanted to understand my options.

The skill told me:
- Try SMOTE for handling imbalance
- Start with Random Forest (handles imbalance better than SVM)
- Use stratified cross-validation
- Consider class weights if SMOTE doesn't work

That 5-minute conversation prevented me from building the wrong pipeline.

### Code Building Mode: The Implementation Phase

Once I know what I want, I switch modes:

```
/scientific-skills:scanpy

Build a complete scRNA-seq analysis pipeline that:
- Loads 10x data from ./data/raw/
- Performs QC (filter cells <200 genes, genes in <3 cells)
- Normalizes and finds top 2000 variable genes
- Runs PCA, UMAP, and Leiden clustering
- Saves figures to ./output/

Following Scanpy best practices.
```

Same skill, totally different intent. This time I get production-ready code with proper error handling, file I/O, and visualization.

The mistake I kept making? Mixing modes. I'd ask for advice and get code, or ask for code and get a lecture. Being explicit about which mode you want fixes this entirely.

## The Catalog: What I Actually Use

169 skills is overwhelming. Here's what I've actually found useful in my day-to-day work:

### For Data Science Work

**scikit-learn (3.8k tokens)** - My most-used skill. Great for "which algorithm should I try first" questions. I invoke it in Q&A mode constantly.

**exploratory-data-analysis (3.4k)** - This one taught me to stop jumping straight to modeling. Now I run through its checklist before touching any ML: check distributions, look for outliers, verify correlations, plot everything.

**statistical-analysis (4.8k)** - Saved me from choosing the wrong hypothesis test at least five times. "I have 3 groups, non-normal data, what test?" → "Kruskal-Wallis, here's why."

### For Bioinformatics

**scanpy (2.7k)** - Single-cell RNA-seq pipelines. I use this in code mode to generate entire analysis scripts. Beats reading docs for hours.

**gget (6.2k)** - Gene/protein lookups. Works in both modes. "What's the function of BRCA1?" or "Write code to batch-query 50 genes."

**biopython (3.4k)** - Sequence manipulation. Mostly code mode for parsing FASTA files and running local BLAST.

### For Visualization

**seaborn (4.8k)** - "Which plot should I use for this data?" Then it generates the code. Both modes work great.

**scientific-visualization (6.3k)** - When I need publication-quality figures. It knows about journal formatting requirements.

### For Writing

**scientific-writing (5.9k)** - I'm still figuring this one out, but it's helped me structure methods sections better.

**citation-management (7.7k)** - Formats bibliographies. Saves me from manually fixing citation styles.

## Prompt Templates That Actually Work

After 100+ failed prompts, here's what finally clicked:

### For Algorithm Selection

```
/scientific-skills:scikit-learn

[Describe your problem with specifics]
Data: 10k samples, 50 features
Target: Binary classification, 90/10 imbalance
Constraints: Need predictions in <100ms

Don't write code. Compare options and recommend an approach.
```

The key: **specifics + explicit "don't write code."**

### For Implementation

```
/scientific-skills:scanpy

Build a scRNA-seq pipeline:
Input: ./data/raw/ (10x format)
Steps: QC → normalize → HVG → PCA → UMAP → cluster
Output: Figures to ./output/, h5ad to ./results/

Following best practices.
```

The key: **explicit I/O paths + step sequence + "best practices."**

### For Database Queries

```
/scientific-skills:uniprot-database

What are the known functions, domains, and disease associations for BRCA1?
Format as a table.
```

The key: **specific question + desired output format.**

## What I Got Wrong About Token Costs

Skills range from 1.5k to 13k tokens. I used to think "bigger = better." Wrong.

Last week I invoked `treatment-plans` (13k tokens) when I just needed basic clinical decision logic. That's like using a sledgehammer to hang a picture frame.

Now I check the catalog first:
- Simple lookups: Use Context7 (variable, usually <1k)
- Focused workflows: Small skills (1.5k-4k)
- Complex domains: Large skills (6k-13k)

The `treatment-plans` skill is genuinely useful for complex clinical workflows. But if I just need to validate a drug name? Context7 or a smaller database skill.

## The Workflow I Settled On

After all this trial and error, here's my current process:

1. **Start with Q&A mode** to understand options
2. **Use Context7** for syntax lookups during planning
3. **Switch to Code mode** once I know what I want
4. **Chain skills** for multi-step workflows

Example from last Friday:

```
# Step 1: Planning
/scientific-skills:statistical-analysis
I have 3 treatment groups, n≈20 each, non-normal data.
What test should I use?
→ Kruskal-Wallis test

# Step 2: Syntax check
[Use Context7]
Look up scipy.stats.kruskal parameter options
→ Confirmed it returns statistic and p-value

# Step 3: Implementation
/scientific-skills:statsmodels
Build complete analysis:
- Load data from ./data/experiment.csv
- Run Kruskal-Wallis
- If significant, post-hoc Dunn test
- Generate violin plots with significance markers
- Save report to ./results/
```

This took 10 minutes total. Before I learned this workflow? I'd spend 2 hours fumbling through documentation and StackOverflow.

## What I'm Still Figuring Out

Some skills remain mysterious to me:

- **Lab integration skills** (Opentrons, Benchling) - I don't have lab hardware to test these with
- **Quantum computing skills** - Invoked `qiskit` once, got overwhelmed, haven't returned
- **Clinical skills** - The 13k-token ones feel powerful but I'm not confident about medical domain accuracy yet

If you've used these successfully, I'd genuinely love to hear about it.

## The Bottom Line

Here's what this entire exploration taught me:

1. **Context7 for syntax, Skills for strategy** - This rule alone would have saved me weeks
2. **Be explicit about mode** - "Don't write code" or "Build a pipeline" prevent confusion
3. **Specificity matters** - Vague prompts get vague results
4. **Token cost is real** - Check the catalog, don't invoke blindly
5. **Chain skills for complex workflows** - Planning skill → implementation skill works great

I'm not claiming this is the definitive guide. I'm still learning which skills work best for different scenarios. But if you're staring at that list of 169 skills wondering where to start, maybe this saves you some of the confusion I went through.

What's your experience been? Have you found skills I'm underutilizing? I'm always looking to refine this mental model.

---

*Skill counts and token sizes accurate as of December 2024. If you spot errors or have better strategies, let me know.*
