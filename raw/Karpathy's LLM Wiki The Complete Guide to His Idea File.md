---
title: "Karpathy's LLM Wiki: The Complete Guide to His Idea File"
source: "https://antigravity.codes/blog/karpathy-llm-wiki-idea-file"
author:
  - "[[Antigravity.codes]]"
published: 2026-04-04
created: 2026-04-08
description: "Karpathy shared his LLM Wiki idea file as a GitHub gist. We break down every concept, tool, and technique — with implementation examples and code."
tags:
  - "LLM"
---
On April 3, 2026, Andrej Karpathy — co-founder of OpenAI, former AI lead at Tesla, and the person who coined “vibe coding” — posted a tweet titled **“LLM Knowledge Bases”** describing how he now uses LLMs to build personal knowledge wikis instead of just generating code. That tweet went massively viral. The next day, he followed up with something new: an **“idea file”** — a GitHub gist that lays out the complete architecture, philosophy, and tooling behind the concept. We covered the original tweet in our [first article](https://antigravity.codes/blog/karpathy-llm-knowledge-bases). This is the deep dive into the follow-up — every word, every tool, every implementation detail.

Get the latest on AI, LLMs & developer tools

New MCP servers, model updates, and guides like this one — delivered weekly.

### 🎬 Watch the Video Breakdown

![](https://www.youtube.com/watch?v=aGXTV5MTqDY)

Prefer reading? Keep scrolling for the full written guide with code examples.

## 1\. The Viral Moment

The original tweet described Karpathy's shift from spending tokens on code to spending tokens on **knowledge**. He outlined a system where raw source documents (articles, papers, repos, datasets, images) get dropped into a

```
raw/
```
directory, and an LLM incrementally “compiles” them into a structured wiki — a collection of interlinked
```
.md
```
files with summaries, backlinks, and concept articles.

The tweet exploded. Karpathy himself acknowledged it: **“Wow, this tweet went very viral!”** So he did something interesting — instead of just sharing the code or the app, he shared an *idea file*.

The follow-up tweet links to a GitHub gist titled **“LLM Wiki”** — a carefully written document that describes the pattern, architecture, operations, and tooling at a conceptual level. It is not code. It is not an app. It is something new.

Read the Full Gist

Karpathy's complete idea file is available here: [gist.github.com/karpathy/442a6bf555914893e9891c11519de94f](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f). You can copy it directly and paste it to your LLM agent to get started.

## 2\. Idea Files: A New Format for the Agent Era

Karpathy introduces a concept he calls an **“idea file”**. His exact words:

Karpathy's Definition

“The idea of the idea file is that in this era of LLM agents, there is less of a point/need of sharing the specific code/app, you just share the idea, then the other person's agent customizes & builds it for your specific needs.”

This is a subtle but profound shift. Traditionally, when a developer builds something useful, they share the *implementation*: a GitHub repo, a package on npm, a Docker image. The recipient clones it, configures it, and runs it. But in a world where everyone has access to an LLM agent (Claude Code, OpenAI Codex, OpenCode, Cursor, etc.), sharing the *idea* can be more useful than sharing the code.

Why? Because the idea is portable. The code is specific. Karpathy uses Obsidian on macOS with Claude Code. You might use VS Code on Linux with OpenAI Codex. A shared GitHub repo would need to be forked, adapted, and debugged. A shared idea file gets copy-pasted to your agent, and **your agent builds a version customized to your exact setup**.

Karpathy says the gist is “intentionally kept a little bit abstract/vague because there are so many directions to take this in.” That's not a bug — it's the design. The document's last line says it plainly: **“The document's only job is to communicate the pattern. Your LLM can figure out the rest.”**

He also mentions that the gist has a Discussion tab where people can “adjust the idea or contribute their own,” turning it into a collaborative idea space. This is a new kind of open source — not open code, but **open ideas**, designed to be interpreted and instantiated by AI agents.

### How to Use the Idea File

Karpathy says you can **“give this to your agent and it can build you your own LLM wiki and guide you on how to use it.”** In practice, that means:

1. Copy the gist content (the full
	```
	llm-wiki.md
	```
	file)
2. Paste it into your LLM agent's context (Claude Code, Codex, OpenCode, or any agentic IDE)
3. Tell the agent: “Set up an LLM Wiki based on this idea file for \[your topic\]”
4. The agent will create the directory structure, write the schema file, and guide you through first ingestion

EXAMPLE: GIVING THE IDEA FILE TO YOUR AGENT  
  
\# In Claude Code, OpenCode, or any agentic IDE:  
  
\> Here is an idea file from Karpathy about building  
\> an LLM Wiki. I want to build one for \[machine learning  
\> research / competitive analysis / book notes / etc.\].  
\>  
\> \[paste the full gist content\]  
\>  
\> Please set up the directory structure, create the  
\> schema file (CLAUDE.md or AGENTS.md), and walk me  
\> through ingesting my first source document.  

## 3\. The Core Idea: Wiki Beats RAG

The heart of the gist is a comparison between how most people use LLMs with documents today versus what Karpathy proposes. Let's break this down precisely.

### The RAG Problem

Karpathy writes: **“Most people's experience with LLMs and documents looks like RAG: you upload a collection of files, the LLM retrieves relevant chunks at query time, and generates an answer.”**

RAG (Retrieval-Augmented Generation) is the dominant pattern for connecting LLMs to private data. Tools like **NotebookLM**, ChatGPT file uploads, and most enterprise AI tools work this way. You upload documents. When you ask a question, the system searches for relevant chunks, feeds them to the LLM, and generates an answer.

The problem, as Karpathy identifies it: **“The LLM is rediscovering knowledge from scratch on every question. There's no accumulation.”**

Ask a question that requires synthesizing five documents, and the RAG system has to find and piece together the relevant fragments every time. Ask the same question tomorrow, and it does the same work again. Nothing is built up. Nothing compounds.

### The Wiki Solution

Karpathy's alternative: **“Instead of just retrieving from raw documents at query time, the LLM incrementally builds and maintains a persistent wiki — a structured, interlinked collection of markdown files that sits between you and the raw sources.”**

When you add a new source, the LLM doesn't just index it for later retrieval. It reads it, extracts key information, and **integrates it into the existing wiki** — updating entity pages, revising topic summaries, noting where new data contradicts old claims, strengthening or challenging the evolving synthesis.

The key line: **“The knowledge is compiled once and then kept current, not re-derived on every query.”**

| Dimension | Traditional RAG | LLM Wiki |
| --- | --- | --- |
| **When knowledge is processed** | At query time (on every question) | At ingest time (once per source) |
| **Cross-references** | Discovered ad-hoc per query | Pre-built and maintained |
| **Contradictions** | May not be noticed | Flagged during ingestion |
| **Knowledge accumulation** | None — starts fresh each query | Compounds with every source and query |
| **Output format** | Chat responses (ephemeral) | Persistent markdown files (durable) |
| **Who maintains it** | The system (black box) | The LLM (transparent, editable) |
| **Human role** | Upload and query | Curate, explore, and question |
| **Examples** | NotebookLM, ChatGPT uploads | Karpathy's LLM Wiki pattern |

### The Compounding Effect

Karpathy emphasizes this repeatedly: **“The wiki is a persistent, compounding artifact.”** The cross-references are already there. The contradictions have already been flagged. The synthesis already reflects everything you've read. Every source you add and every question you ask makes the wiki richer.

### The Human-LLM Division of Labor

Karpathy's description of the workflow: **“You never (or rarely) write the wiki yourself — the LLM writes and maintains all of it. You're in charge of sourcing, exploration, and asking the right questions. The LLM does all the grunt work — the summarizing, cross-referencing, filing, and bookkeeping.”**

His daily setup: **“I have the LLM agent open on one side and Obsidian open on the other. The LLM makes edits based on our conversation, and I browse the results in real time — following links, checking the graph view, reading the updated pages.”**

Then the analogy that captures the whole system: **“Obsidian is the IDE; the LLM is the programmer; the wiki is the codebase.”**

## 4\. The Three-Layer Architecture

Karpathy defines three distinct layers. Each has a clear owner and a clear purpose.

### Layer 1: Raw Sources

**Karpathy writes:** “Your curated collection of source documents. Articles, papers, images, data files. These are immutable — the LLM reads from them but never modifies them. This is your source of truth.”

The

```
raw/
```
directory is sacred. The LLM can read anything in it but must never write to it. This is critical because it means you always have the original sources to verify against. If the LLM makes a mistake in the wiki, you can trace back to the raw source and correct it.

RAW SOURCES DIRECTORY EXAMPLE  
  
raw/  
articles/  
2026-03-attention-is-all-you-need-revisited.md  
2026-04-scaling-laws-update.md  
papers/  
transformer-architecture-v2.pdf  
mixture-of-experts-survey.pdf  
repos/  
llama-3-readme.md  
vllm-architecture-notes.md  
data/  
benchmark-results.csv  
model-comparison.json  
images/  
transformer-diagram.png  
scaling-curves.png  
assets/  
\# Downloaded images from clipped articles  

### Layer 2: The Wiki

**Karpathy writes:** “A directory of LLM-generated markdown files. Summaries, entity pages, concept pages, comparisons, an overview, a synthesis. The LLM owns this layer entirely. It creates pages, updates them when new sources arrive, maintains cross-references, and keeps everything consistent. You read it; the LLM writes it.”

WIKI DIRECTORY EXAMPLE  
  
wiki/  
index.md # Master catalog of all pages  
log.md # Chronological activity record  
overview.md # High-level synthesis  
concepts/  
attention-mechanism.md  
mixture-of-experts.md  
scaling-laws.md  
tokenization.md  
entities/  
openai.md  
anthropic.md  
google-deepmind.md  
sources/  
summary-attention-revisited.md  
summary-scaling-update.md  
comparisons/  
gpt4-vs-claude-vs-gemini.md  
rag-vs-finetuning.md  

The key insight: the wiki sits **between** you and the raw sources. You don't read raw papers to answer questions — you read the wiki. The wiki is pre-digested, cross-referenced, and synthesized. It's what a research assistant would produce if they read everything and organized it for you.

### Layer 3: The Schema

**Karpathy writes:** “A document (e.g. CLAUDE.md for Claude Code or AGENTS.md for Codex) that tells the LLM how the wiki is structured, what the conventions are, and what workflows to follow when ingesting sources, answering questions, or maintaining the wiki. This is the key configuration file — it's what makes the LLM a disciplined wiki maintainer rather than a generic chatbot.”

The schema is the most important piece. Without it, the LLM is just a chatbot that happens to have access to files. With it, the LLM becomes a **systematic wiki maintainer** that follows consistent rules across sessions.

Karpathy adds: **“You and the LLM co-evolve this over time as you figure out what works for your domain.”** The schema isn't static. You start with something basic and refine it as you learn what page structures, frontmatter fields, and workflows work best.

EXAMPLE SCHEMA: CLAUDE.md (for Claude Code)  
  
\# LLM Wiki Schema  
  
\## Project Structure  
\- \`raw/\` — immutable source documents. NEVER modify.  
\- \`wiki/\` — LLM-generated wiki. You own this entirely.  
\- \`wiki/index.md\` — master catalog. Update on every ingest.  
\- \`wiki/log.md\` — append-only activity log.  
  
\## Page Conventions  
Every wiki page MUST have YAML frontmatter:  
\`\`\`  
\---  
title: Page Title  
type: concept | entity | source-summary | comparison  
sources: \[list of raw/ files referenced\]  
related: \[list of wiki pages linked\]  
created: YYYY-MM-DD  
updated: YYYY-MM-DD  
confidence: high | medium | low  
\---  
\`\`\`  
  
\## Ingest Workflow  
When I say "ingest \[filename\]":  
1\. Read the source file in raw/  
2\. Discuss key takeaways with me  
3\. Create/update a summary page in wiki/sources/  
4\. Update wiki/index.md  
5\. Update all relevant concept and entity pages  
6\. Append an entry to wiki/log.md  
  
\## Query Workflow  
When I ask a question:  
1\. Read wiki/index.md to find relevant pages  
2\. Read those pages  
3\. Synthesize an answer with \[\[wiki-link\]\] citations  
4\. If the answer is valuable, offer to file it as  
a new wiki page  
  
\## Lint Workflow  
When I say "lint":  
1\. Check for contradictions between pages  
2\. Find orphan pages with no inbound links  
3\. List concepts mentioned but lacking own page  
4\. Check for stale claims superseded by newer sources  
5\. Suggest questions to investigate next  

If you're using **OpenAI Codex**, the same schema goes into

```
AGENTS.md
```
instead. If you're using **OpenCode**, it goes in
```
OPENCODE.md
```
. The content is the same — only the filename changes based on which agent reads it.

Why the Schema Matters

Without a schema, every session with the LLM starts from zero. The LLM doesn't know your conventions, your page formats, or your workflows. You end up re-explaining everything. The schema is persistent memory — it carries knowledge across sessions and ensures consistency. It's what turns a generic LLM into **your** wiki maintainer.

## 5\. Operations: Ingest, Query, Lint

Karpathy defines three core operations. Each one has a clear trigger, a clear process, and a clear output.

### Operation 1: Ingest

**Karpathy writes:** “You drop a new source into the raw collection and tell the LLM to process it. An example flow: the LLM reads the source, discusses key takeaways with you, writes a summary page in the wiki, updates the index, updates relevant entity and concept pages across the wiki, and appends an entry to the log. A single source might touch 10-15 wiki pages.”

This is the most important operation. A single ingest doesn't just create one new page — it ripples across the entire wiki. If you ingest a paper about a new transformer variant, the LLM might:

- Create a new summary page for the paper
- Update the “Attention Mechanism” concept page with the new variant
- Update the “Scaling Laws” page if the paper contains new benchmarks
- Update the entity page for the paper's authors or their organization
- Update the comparison page if the paper benchmarks against known models
- Add links from existing pages that now reference the new content
- Update the index with the new page
- Log the ingest in the activity log

Karpathy adds a personal preference: **“I prefer to ingest sources one at a time and stay involved — I read the summaries, check the updates, and guide the LLM on what to emphasize. But you could also batch-ingest many sources at once with less supervision.”**

EXAMPLE INGEST SESSION  
  
\# You drop a new article into raw/  
  
\> I just added a new article to raw/articles/  
\> 2026-04-mixture-of-experts-efficiency.md  
\> Please ingest it.  
  
\# The LLM reads it, then responds:  
  
Key takeaways from this article:  
1\. MoE models achieve 4x throughput at same quality  
2\. Routing efficiency improved 23% over Switch Transformer  
3\. Contradicts our wiki claim that dense models are  
always more efficient below 10B params  
  
I will:  
\- Create wiki/sources/summary-moe-efficiency-2026.md  
\- Update wiki/concepts/mixture-of-experts.md  
\- Update wiki/concepts/scaling-laws.md (new benchmarks)  
\- Flag contradiction on wiki/concepts/dense-vs-sparse.md  
\- Update wiki/index.md  
\- Append to wiki/log.md  
  
Shall I proceed?  

### Operation 2: Query

**Karpathy writes:** “You ask questions against the wiki. The LLM searches for relevant pages, reads them, and synthesizes an answer with citations. Answers can take different forms depending on the question — a markdown page, a comparison table, a slide deck (Marp), a chart (matplotlib), a canvas.”

But the most important insight about queries is this: **“Good answers can be filed back into the wiki as new pages.”** A comparison you asked for, an analysis, a connection you discovered — these are valuable and shouldn't disappear into chat history. **“This way your explorations compound in the knowledge base just like ingested sources do.”**

This is the compounding loop: sources get ingested into the wiki, queries generate new insights, and the best insights get filed back as wiki pages. The wiki grows not just from external sources, but from your own exploration.

EXAMPLE QUERY WITH FILING  
  
\> Compare the routing strategies in MoE models across  
\> our sources. Which approach has the best efficiency/  
\> quality tradeoff?  
  
\# LLM reads wiki/concepts/mixture-of-experts.md,  
\# wiki/sources/summary-moe-\*.md, produces analysis  
  
\[LLM produces a detailed comparison with a table\]  
  
\> This is great. File it as a wiki page.  
  
\# LLM creates wiki/comparisons/moe-routing-strategies.md  
\# Updates index.md, adds links from related pages  

### Operation 3: Lint

**Karpathy writes:** “Periodically, ask the LLM to health-check the wiki. Look for: contradictions between pages, stale claims that newer sources have superseded, orphan pages with no inbound links, important concepts mentioned but lacking their own page, missing cross-references, data gaps that could be filled with a web search.”

He adds: **“The LLM is good at suggesting new questions to investigate and new sources to look for. This keeps the wiki healthy as it grows.”**

EXAMPLE LINT OUTPUT  
  
\> Lint the wiki.  
  
Wiki Health Report (2026-04-04):  
  
CONTRADICTIONS (2):  
\- concepts/dense-vs-sparse.md claims dense > sparse  
below 10B, but sources/summary-moe-efficiency.md  
shows opposite. Recommend: update with nuance.  
\- entities/openai.md says GPT-5 is 200B params,  
but sources/summary-gpt5-leak.md says 300B.  
  
ORPHAN PAGES (3):  
\- concepts/tokenization.md (no inbound links)  
\- sources/summary-old-bert-paper.md (no references)  
\- comparisons/old-gpu-benchmark.md (outdated)  
  
MISSING PAGES (4):  
\- "RLHF" mentioned 12 times, no concept page  
\- "Constitutional AI" mentioned 8 times, no page  
\- "KV Cache" referenced in 5 sources, no page  
\- "Speculative Decoding" mentioned 3 times, no page  
  
SUGGESTED INVESTIGATIONS:  
\- No sources on inference optimization post-2025  
\- Entity page for Meta AI is thin (only 1 source)  
\- The "Scaling Laws" page hasn't been updated in 3 weeks  

## 6\. Indexing and Logging

Karpathy defines two special files that are critical to how the LLM navigates the wiki. They serve different purposes and both are important.

### index.md: The Content Catalog

**Karpathy writes:** “index.md is content-oriented. It's a catalog of everything in the wiki — each page listed with a link, a one-line summary, and optionally metadata like date or source count. Organized by category (entities, concepts, sources, etc.). The LLM updates it on every ingest.”

The key insight about index.md is how it replaces RAG: **“When answering a query, the LLM reads the index first to find relevant pages, then drills into them. This works surprisingly well at moderate scale (~100 sources, ~hundreds of pages) and avoids the need for embedding-based RAG infrastructure.”**

This is a practical revelation. Most people assume you need vector databases and embedding pipelines for knowledge retrieval. Karpathy says: at moderate scale, a well-maintained index file is enough. The LLM reads the index (a few thousand tokens), identifies relevant pages, and reads those directly.

EXAMPLE: wiki/index.md  
  
\# Wiki Index  
  
\## Concepts  
\- \[\[attention-mechanism\]\] — Self-attention, multi-head  
attention, and variants (12 sources)  
\- \[\[mixture-of-experts\]\] — Sparse MoE architectures,  
routing strategies (8 sources)  
\- \[\[scaling-laws\]\] — Chinchilla, Kaplan laws,  
compute-optimal training (15 sources)  
\- \[\[tokenization\]\] — BPE, SentencePiece, tiktoken  
(3 sources)  
  
\## Entities  
\- \[\[openai\]\] — GPT series, organizational history  
(20 sources)  
\- \[\[anthropic\]\] — Claude series, constitutional AI  
(14 sources)  
\- \[\[google-deepmind\]\] — Gemini, PaLM, AlphaFold  
(18 sources)  
  
\## Source Summaries  
\- \[\[summary-attention-revisited\]\] — 2026-03-15  
\- \[\[summary-moe-efficiency\]\] — 2026-04-01  
\- \[\[summary-scaling-update\]\] — 2026-04-02  
  
\## Comparisons  
\- \[\[moe-routing-strategies\]\] — Filed from query 2026-04-04  
\- \[\[rag-vs-finetuning\]\] — Tradeoffs and use cases  

### log.md: The Activity Timeline

**Karpathy writes:** “log.md is chronological. It's an append-only record of what happened and when — ingests, queries, lint passes.”

He includes a practical tip: **“If each entry starts with a consistent prefix (e.g.**

```
## [2026-04-02] ingest | Article Title
```
**), the log becomes parseable with simple unix tools —
```
grep "^## [" log.md | tail -5
```
gives you the last 5 entries.”**

EXAMPLE: wiki/log.md  
  
\# Activity Log  
  
\## \[2026-04-04\] ingest | MoE Efficiency Article  
Source: raw/articles/2026-04-mixture-of-experts-efficiency.md  
Pages created: sources/summary-moe-efficiency.md  
Pages updated: concepts/mixture-of-experts.md,  
concepts/scaling-laws.md, concepts/dense-vs-sparse.md  
Notes: Contradicts dense-vs-sparse claim below 10B params.  
Flagged for review.  
  
\## \[2026-04-04\] query | MoE Routing Comparison  
Question: Compare routing strategies in MoE models  
Pages read: concepts/mixture-of-experts.md, 3 source summaries  
Output: Filed as comparisons/moe-routing-strategies.md  
  
\## \[2026-04-04\] lint | Weekly Health Check  
Contradictions found: 2  
Orphan pages: 3  
Missing pages suggested: 4  
Investigations suggested: 3  
  
\## \[2026-04-03\] ingest | Scaling Laws Update  
Source: raw/articles/2026-04-scaling-laws-update.md  
Pages created: sources/summary-scaling-update.md  
Pages updated: concepts/scaling-laws.md, entities/openai.md  

The log also helps the LLM understand what's been done recently. When you start a new session, the LLM can read the last few log entries to understand the current state of the wiki.

## 7\. The Tool Stack

Karpathy mentions several specific tools in the gist. Here's what each one does and how it fits into the workflow.

### qmd: Local Search for Markdown

**Karpathy writes:** “ [qmd](https://github.com/tobi/qmd) is a good option: it's a local search engine for markdown files with hybrid BM25/vector search and LLM re-ranking, all on-device. It has both a CLI (so the LLM can shell out to it) and an MCP server (so the LLM can use it as a native tool).”

**qmd** was built by Tobi Lutke, CEO of Shopify. It's designed exactly for the use case Karpathy describes: searching over a collection of markdown files. It combines three search strategies:

- **BM25 full-text search** — keyword matching (fast, precise)
- **Vector semantic search** — meaning-based matching (finds related concepts)
- **LLM re-ranking** — the LLM scores results for relevance (highest quality)

Everything runs locally via

```
node-llama-cpp
```
with GGUF models. No cloud API calls. No data leaves your machine.

GETTING STARTED WITH QMD  
  
\# Install qmd globally  
npm install -g @tobilu/qmd  
  
\# Add your wiki as a collection  
qmd collection add./wiki --name my-research  
  
\# Keyword search (BM25)  
qmd search "mixture of experts routing"  
  
\# Semantic search (vector)  
qmd vsearch "how do sparse models handle efficiency"  
  
\# Hybrid search with LLM re-ranking (best quality)  
qmd query "what are the tradeoffs of top-k vs expert-choice routing"  
  
\# JSON output for piping to LLM agents  
qmd query "scaling laws" --json  
  
\# Start qmd as an MCP server for Claude Code / etc.  
qmd mcp  

Karpathy notes that at small scale, the

```
index.md
```
file is enough for navigation. **qmd becomes useful as the wiki grows beyond what the index can handle** — likely once you have hundreds of pages and the index itself is too large to read in one context window.

### Obsidian Web Clipper

**Karpathy writes:** “Obsidian Web Clipper is a browser extension that converts web articles to markdown. Very useful for quickly getting sources into your raw collection.”

The [Web Clipper](https://obsidian.md/clipper) is available for Chrome, Firefox, Safari, Edge, Brave, and Arc. When you clip an article, it:

- Converts the HTML to clean markdown
- Adds YAML frontmatter (author, date, source URL, tags)
- Preserves formatting, code blocks, and images
- Saves directly to your Obsidian vault (your
	```
	raw/
	```
	directory)

It also supports **templates** — you can define different clipping formats for articles, recipes, academic papers, or any other content type. This makes ingestion consistent and predictable.

### Downloading Images Locally

Karpathy gives a specific tip for images: **“In Obsidian Settings → Files and links, set ‘Attachment folder path’ to a fixed directory (e.g.**

```
raw/assets/
```
**). Then in Settings → Hotkeys, search for ‘Download’ to find ‘Download attachments for current file’ and bind it to a hotkey (e.g. Ctrl+Shift+D).”**

After clipping an article, you hit the hotkey and all images get downloaded to local disk. Why does this matter? Because it **“lets the LLM view and reference images directly instead of relying on URLs that may break.”**

He also notes a current limitation: **“LLMs can't natively read markdown with inline images in one pass — the workaround is to have the LLM read the text first, then view some or all of the referenced images separately to gain additional context.”**

### Obsidian's Graph View

**Karpathy writes:** “Obsidian's graph view is the best way to see the shape of your wiki — what's connected to what, which pages are hubs, which are orphans.”

The graph view renders all your wiki pages as nodes and all

```
[[wiki-links]]
```
as edges. Hub pages (like core concepts with many connections) appear as large nodes. Orphan pages (with no links) appear isolated. This gives you an immediate visual sense of where your knowledge is dense and where there are gaps.

### Marp: Markdown Slide Decks

**Karpathy writes:** “Marp is a markdown-based slide deck format. Obsidian has a plugin for it. Useful for generating presentations directly from wiki content.”

[Marp](https://marp.app/) lets you write presentations in pure Markdown. You separate slides with

```
---
```
(horizontal rules). It supports themes, image syntax, math typesetting, and exports to HTML, PDF, and PowerPoint.

EXAMPLE MARP SLIDE DECK (generated by LLM from wiki)  
  
\---  
marp: true  
theme: default  
\---  
  
\# Mixture of Experts: Key Findings  
  
Compiled from 8 sources in the research wiki  
  
\---  
  
\## Routing Strategies Compared  
  
| Strategy | Throughput | Quality |  
|----------|-----------|---------|  
| Top-K | 2.1x | Baseline |  
| Expert Choice | 3.4x | +2% |  
| Hash | 4.0x | -1% |  
  
\---  
  
\## Key Insight  
  
Expert-choice routing gives the best quality/efficiency  
tradeoff for models above 10B parameters.  
  
Source: wiki/comparisons/moe-routing-strategies.md  

### Dataview: Query Your Frontmatter

**Karpathy writes:** “Dataview is an Obsidian plugin that runs queries over page frontmatter. If your LLM adds YAML frontmatter to wiki pages (tags, dates, source counts), Dataview can generate dynamic tables and lists.”

[Dataview](https://blacksmithgu.github.io/obsidian-dataview/) treats your vault as a database. If your wiki pages have frontmatter like

```
type: concept
```
,
```
sources: [file1, file2]
```
,
```
confidence: high
```
, then Dataview lets you query it with an SQL-like language:

DATAVIEW QUERY EXAMPLES  
  
\# List all concept pages with source counts  
\`\`\`dataview  
TABLE length(sources) AS "Sources", confidence  
FROM "wiki/concepts"  
SORT length(sources) DESC  
\`\`\`  
  
\# Find pages updated in the last week  
\`\`\`dataview  
LIST  
FROM "wiki"  
WHERE updated >= date(today) - dur(7 days)  
SORT updated DESC  
\`\`\`  
  
\# Find low-confidence pages that need review  
\`\`\`dataview  
TABLE title, sources  
FROM "wiki"  
WHERE confidence = "low"  
SORT file.name ASC  
\`\`\`  

### Git: Version Control for Knowledge

**Karpathy writes:** “The wiki is just a git repo of markdown files. You get version history, branching, and collaboration for free.”

This is understated but powerful. Because the entire wiki is plain markdown in a directory, you can:

- ```
	git log
	```
	to see how the wiki evolved over time
- ```
	git diff
	```
	to see exactly what changed in each ingest
- ```
	git revert
	```
	to roll back a bad compilation
- ```
	git branch
	```
	to explore alternative organizational structures
- ```
	git blame
	```
	to trace when a specific claim was added
- Use GitHub/GitLab for team collaboration with pull requests

| Tool | Role in LLM Wiki | Required? |
| --- | --- | --- |
| **Obsidian** | IDE / viewer for browsing the wiki | Recommended (any markdown viewer works) |
| **Obsidian Web Clipper** | Ingestion: clip web articles to markdown | Recommended for web sources |
| **qmd** | Search engine for large wikis | Optional (index.md works at small scale) |
| **Marp** | Output: generate slide decks from wiki | Optional |
| **Dataview** | Query frontmatter for dashboards | Optional |
| **Git** | Version control for the wiki | Recommended |
| **LLM Agent** | Wiki maintainer (Claude Code, Codex, etc.) | Required |

## 8\. Use Cases Karpathy Lists

The gist lists five specific contexts where this pattern applies. Let's look at each one with implementation details.

### Personal Knowledge Base

**Karpathy writes:** “Tracking your own goals, health, psychology, self-improvement — filing journal entries, articles, podcast notes, and building up a structured picture of yourself over time.”

Implementation: Create a personal wiki with sections for goals, health metrics, reading notes, and reflections. Ingest journal entries, articles you read, podcast transcripts. The LLM builds concept pages for recurring themes (“sleep quality,” “exercise routine,” “career goals”) and connects them across time. Ask questions like: “What patterns do I see in my energy levels over the last 3 months?”

### Research

**Karpathy writes:** “Going deep on a topic over weeks or months — reading papers, articles, reports, and incrementally building a comprehensive wiki with an evolving thesis.”

This is Karpathy's primary use case. His research wiki has ~100 articles and ~400,000 words on a single ML research topic. The wiki builds an evolving thesis that gets refined with every new source.

### Reading a Book

**Karpathy writes:** “Filing each chapter as you go, building out pages for characters, themes, plot threads, and how they connect. By the end you have a rich companion wiki.”

He uses a vivid example: **“Think of fan wikis like [Tolkien Gateway](https://tolkiengateway.net/wiki/Main_Page) — thousands of interlinked pages covering characters, places, events, languages, built by a community of volunteers over years. You could build something like that personally as you read, with the LLM doing all the cross-referencing and maintenance.”**

Imagine reading *War and Peace*. After each chapter, you ingest your notes. The LLM maintains character pages (tracking their development across chapters), theme pages (connecting recurring ideas), and a timeline page. By the end, you have a personal companion wiki that rivals a literary analysis.

### Business / Team

**Karpathy writes:** “An internal wiki maintained by LLMs, fed by Slack threads, meeting transcripts, project documents, customer calls. Possibly with humans in the loop reviewing updates. The wiki stays current because the LLM does the maintenance that no one on the team wants to do.”

This is the enterprise version. The sources are internal: Slack exports, meeting recordings (transcribed), project docs, customer call logs, CRM data. The wiki compiles decision logs, project timelines, customer insights, and team knowledge. The human-in-the-loop reviews updates before they become part of the wiki.

### Everything Else

**Karpathy writes:** “Competitive analysis, due diligence, trip planning, course notes, hobby deep-dives — anything where you're accumulating knowledge over time and want it organized rather than scattered.”

The pattern is universal: if you're collecting information from multiple sources over time and want it structured, an LLM Wiki applies. We covered detailed implementations for competitive intelligence, legal compliance, academic literature reviews, and more in our [previous article](https://antigravity.codes/blog/karpathy-llm-knowledge-bases).

## 9\. Step-by-Step Implementation Guide

Here is how to build a working LLM Wiki from scratch, following Karpathy's architecture exactly.

### Step 1: Set Up the Directory Structure

TERMINAL  
  
mkdir -p my-research/raw/articles  
mkdir -p my-research/raw/papers  
mkdir -p my-research/raw/repos  
mkdir -p my-research/raw/assets  
mkdir -p my-research/wiki/concepts  
mkdir -p my-research/wiki/entities  
mkdir -p my-research/wiki/sources  
mkdir -p my-research/wiki/comparisons  
touch my-research/wiki/index.md  
touch my-research/wiki/log.md  
touch my-research/wiki/overview.md  
  
\# Initialize git  
cd my-research && git init  
  
\# Open in Obsidian as a vault  

### Step 2: Create the Schema File

Create a

```
CLAUDE.md
```
(for Claude Code),
```
AGENTS.md
```
(for Codex), or equivalent schema file at the root of your project. Use the example schema from Section 4 above as a starting point. Customize it for your domain.

### Step 3: Configure Obsidian

1. **Install Obsidian** and open
	```
	my-research/
	```
	as a vault
2. **Install Web Clipper** browser extension
3. Settings → Files and links → Set “Attachment folder path” to
	```
	raw/assets
	```
4. Settings → Hotkeys → Bind “Download attachments for current file” to
	```
	Ctrl+Shift+D
	```
5. **Install Marp Slides plugin** (optional, for presentations)
6. **Install Dataview plugin** (optional, for frontmatter queries)

### Step 4: Ingest Your First Source

1. Clip a web article using Web Clipper → save to
	```
	raw/articles/
	```
2. Hit
	```
	Ctrl+Shift+D
	```
	to download images locally
3. Open your LLM agent (Claude Code, Codex, OpenCode, etc.)
4. Tell it: “Ingest raw/articles/\[filename\].md”
5. Review the summary, guide emphasis, approve the wiki updates
6. Check the wiki in Obsidian — browse the new pages, check the graph view
7. Commit:
	```
	git add . && git commit -m “ingest: [article title]”
	```

### Step 5: Build Up Over Time

Repeat the ingest process for each new source. After 10-20 sources, start querying the wiki. After 50+, consider adding **qmd** for search. Run lint checks weekly.

The 10-Source Test

Start with just 10 sources on one topic. Ingest them all. Then ask the wiki a question that requires synthesizing multiple sources. If the structured wiki gives you an insight you wouldn't have gotten by reading the sources individually, the system is working. Scale from there.

### Step 6: Evolve the Schema

As you use the wiki, you'll discover what works and what doesn't. Update the schema (CLAUDE.md / AGENTS.md) accordingly. Maybe you need a new page type. Maybe your frontmatter needs more fields. Maybe your ingest workflow should include a step you didn't anticipate. Karpathy says: **“You and the LLM co-evolve this over time.”**

## 10\. The Memex Connection (1945)

Karpathy closes the gist with a historical connection that puts the whole idea in perspective:

Karpathy's Words

“The idea is related in spirit to Vannevar Bush's Memex (1945) — a personal, curated knowledge store with associative trails between documents. Bush's vision was closer to this than to what the web became: private, actively curated, with the connections between documents as valuable as the documents themselves. The part he couldn't solve was who does the maintenance. The LLM handles that.”

In 1945, **Vannevar Bush** — an MIT engineer who directed the US Office of Scientific Research and Development — published an article in *The Atlantic* called [“As We May Think”](https://www.theatlantic.com/magazine/archive/1945/07/as-we-may-think/303881/). He described a hypothetical device called the **Memex** (memory + index): a desk-sized machine where an individual could store all their books, records, and communications on microfilm, search them rapidly, and create **associative trails** — linked sequences of documents with personal annotations.

Bush's key insight was that the human mind works by **association**, not alphabetical order. Hierarchical filing systems (like library catalogs) force you into rigid categories. The Memex would let you create your own paths through knowledge — linking a chemistry paper to an economics report to a historical essay, following your own logic.

His famous quote: **“Wholly new forms of encyclopedias will appear, ready-made with a mesh of associative trails running through them.”**

The Memex directly inspired:

- **Douglas Engelbart** — who read Bush's article in 1945, “became infected with the idea,” and went on to invent the computer mouse and the concept of personal computing
- **Ted Nelson** — who coined the term “hypertext” in 1965, directly inspired by the Memex's associative trails
- **Tim Berners-Lee** — whose World Wide Web (1989) implemented hypertext at global scale

But as Karpathy observes, the web became *public and chaotic* rather than *private and curated*. Bush imagined something personal — your knowledge, your connections, your trails. The LLM Wiki is closer to that original vision. It's private, actively curated, and the connections between documents are as valuable as the documents themselves.

The missing piece that Bush couldn't solve in 1945: **who does the maintenance?** Creating associative trails, updating connections, keeping everything consistent — that's tedious, manual work. Humans abandon knowledge systems because the maintenance burden grows faster than the value. As Karpathy writes: **“LLMs don't get bored, don't forget to update a cross-reference, and can touch 15 files in one pass. The wiki stays maintained because the cost of maintenance is near zero.”**

## 11\. Community Ideas from the Gist

The GitHub gist has a Discussion tab that Karpathy specifically called out: “People can adjust the idea or contribute their own in the Discussion which is cool.” Here are some notable contributions from the community:

### The.brain Folder Pattern

A developer shared a related pattern: a

```
.brain
```
folder at the root of a project containing markdown files (
```
index.md
```
,
```
architecture.md
```
,
```
decisions.md
```
,
```
changelog.md
```
,
```
deployment.md
```
) that acts as persistent memory across AI sessions. The core rule: **“Read.brain before making changes. Update.brain after making changes. Never commit it to git.”** This is a lighter-weight version of Karpathy's schema — project-specific rather than knowledge-base-specific.

### Inter-Agent Communication via Gists

Another contributor described using GitHub gists as communication channels between different AI agents. Mid-development, they push gists with diagrams (as SVGs) and context, then pass them between different AI frontends (Claude, Grok, etc.). This extends Karpathy's idea file concept — gists become not just human-to-agent communication, but **agent-to-agent communication**.

### The Append-and-Review Note

A community member pointed out that Karpathy's earlier blog post from 2025, “The Append and Review Note” (on [karpathy.bearblog.dev](https://karpathy.bearblog.dev/)), feels like it should be part of this pattern. That post described a simpler workflow: an append-only notes file that gets periodically reviewed and reorganized. The LLM Wiki is the evolved version — the LLM does the review and reorganization automatically.

### Team Knowledge Sharing

One question from the community: “How can I share the knowledge base with my team? Currently we create a RAG and then an MCP server.” Since the wiki is just a git repo, the natural answer is: push it to a shared repository. Team members can browse it in Obsidian, and the LLM agent can be configured to accept updates from multiple contributors. The schema file defines the rules; Git handles collaboration.

## 12\. What This Means

### The “Idea File” as a New Open Source Format

Karpathy may have accidentally created a new format for sharing ideas in the AI era. Instead of sharing code (which is implementation-specific), you share a structured description of the pattern, designed to be interpreted by an LLM agent. The agent adapts it to the user's environment, tools, and preferences. This is **open ideas** rather than open source.

### Why This Pattern Will Spread

Karpathy explains exactly why wikis maintained by LLMs succeed where human-maintained wikis fail: **“The tedious part of maintaining a knowledge base is not the reading or the thinking — it's the bookkeeping. Updating cross-references, keeping summaries current, noting when new data contradicts old claims, maintaining consistency across dozens of pages. Humans abandon wikis because the maintenance burden grows faster than the value. LLMs don't get bored, don't forget to update a cross-reference, and can touch 15 files in one pass.”**

### From Karpathy's Tweet to Your Wiki

The gist ends with a deliberate call to action: **“The right way to use this is to share it with your LLM agent and work together to instantiate a version that fits your needs. The document's only job is to communicate the pattern. Your LLM can figure out the rest.”**

That's the whole point. Don't overthink the setup. Don't wait for someone to build the perfect tool. Copy the gist, paste it to your agent, and start with one topic and 10 sources. The LLM will figure out the directory structure, the page formats, the frontmatter schema. You provide the sources and the questions. The wiki builds itself.

The Takeaway

Karpathy's gist is not a blueprint — it's a **seed**. You give it to your LLM agent, and together you grow it into something specific to your domain. The wiki is a persistent, compounding artifact that gets richer with every source and every question. The LLM does all the bookkeeping. You do the thinking.

## 13\. All Resources & Links

Every resource, tool, and reference mentioned in this article and in Karpathy's gist:

### Karpathy's Posts

- [Original tweet: “LLM Knowledge Bases” (Apr 3, 2026)](https://x.com/karpathy/status/2039805659525644595)
- [Follow-up tweet: “Idea File” (Apr 4, 2026)](https://x.com/karpathy/status/2040470801506541998)
- [GitHub Gist: LLM Wiki (the full idea file)](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f)
- [Karpathy's Blog (bearblog)](https://karpathy.bearblog.dev/)

### Tools Mentioned

- [qmd](https://github.com/tobi/qmd) — Local markdown search engine by Tobi Lutke (BM25 + vector + LLM re-ranking)
- [Obsidian](https://obsidian.md/) — Markdown-based knowledge management app
- [Obsidian Web Clipper](https://obsidian.md/clipper) — Browser extension for clipping web articles to markdown
- [Marp](https://marp.app/) — Markdown-based slide deck framework (exports to HTML, PDF, PowerPoint)
- [Dataview](https://blacksmithgu.github.io/obsidian-dataview/) — Obsidian plugin for querying page frontmatter
- [Tolkien Gateway](https://tolkiengateway.net/wiki/Main_Page) — Example of a comprehensive interlinked wiki

### Concepts & History

- [“As We May Think” by Vannevar Bush (1945)](https://www.theatlantic.com/magazine/archive/1945/07/as-we-may-think/303881/) — The Atlantic article that described the Memex
- [Memex (Wikipedia)](https://en.wikipedia.org/wiki/Memex) — History and influence of Bush's concept
- [Google NotebookLM](https://notebooklm.google/) — RAG-based research tool (the approach Karpathy is moving beyond)

### LLM Agent Platforms (for the schema file)

- **Claude Code** — uses
	```
	CLAUDE.md
	```
	for project instructions
- **OpenAI Codex** — uses
	```
	AGENTS.md
	```
	for project instructions
- **OpenCode** — uses
	```
	OPENCODE.md
	```
	for project instructions
- **Cursor, Windsurf, etc.** — each has its own schema file convention

### Our Coverage

- [Part 1: Karpathy's LLM Knowledge Bases — The Post-Code AI Workflow](https://antigravity.codes/blog/karpathy-llm-knowledge-bases) — Coverage of the original viral tweet
- **Part 2: This article** — Deep dive into the follow-up gist and idea file

### Get the Ultimate Antigravity Cheat Sheet

Join 5,000+ developers and get our exclusive PDF guide to mastering Gemini 3 shortcuts and agent workflows.