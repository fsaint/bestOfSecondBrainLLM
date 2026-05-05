# Best Tools for LLM Second Brains / Long-Term Memory

I've been researching tools and patterns for giving LLMs persistent, long-term memory: a "second brain" that survives across sessions. The problem: a single storage strategy (pure RAG, pure markdown, pure graph) always loses something. The best systems combine multiple representations.

**What are you using? What's missing from this list?**

---

## The Pattern (Karpathy's LLM Wiki)

Before the tools — the conceptual foundation comes from [Andrej Karpathy's LLM Wiki gist](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f):

> Instead of repeatedly searching documents at query time (RAG), compile knowledge once into a structured, interlinked markdown wiki. The wiki is a persistent, compounding artifact — cross-references are already there, synthesis already reflects everything you've read.

Three layers: **raw sources** (immutable originals) → **wiki** (LLM-maintained markdown) → **schema** (AGENTS.md / CLAUDE.md conventions). This gist spawned a whole ecosystem.

---

## Tools

### 📁 Markdown Vault Managers

**[Tolaria](https://github.com/refactoringhq/tolaria)** — Desktop app (Tauri + React) for managing markdown knowledge bases. Files-first, git-first, offline-first. No database — the filesystem is the source of truth. Uses YAML frontmatter conventions (`type:`, `belongs_to:`, `related_to:`, `has:`) and wiki-style `[[wikilinks]]` for relationships. Ships an MCP server so AI agents can `search_notes`, `get_note`, and `get_vault_context`. The creator runs his entire life on a 10,000+ note vault.

**[Obsidian Skills](https://github.com/kepano/obsidian-skills)** — Agent skills (Claude Code, Codex, OpenCode, Cursor) for working with Obsidian vaults. Covers Obsidian Flavored Markdown, wikilinks, frontmatter, embeds, callouts, and Bases. The `defuddle` skill is particularly useful: extracts clean markdown from any web URL (`defuddle parse <url> --md`), removing navigation and clutter before ingestion.

---

### 🔍 Local Search Engines

**[QMD](https://github.com/tobi/qmd)** — On-device search engine for markdown docs. Combines **BM25 + vector semantic search + LLM reranking**, all running locally via GGUF models. Collections map to folders. MCP server exposes `query`, `get`, `multi_get`, `status`. TypeScript SDK for programmatic use. Referenced by Karpathy as the recommended local search layer. SQLite-backed — no external server needed. Run as an HTTP daemon to keep models loaded in VRAM.

```bash
qmd collection add ~/notes --name notes
qmd embed
qmd query "how did we decide on the pricing model?"   # hybrid + reranking
qmd mcp --http --daemon                               # persistent MCP server
```

---

### 🕸️ Knowledge Graphs

**[Graphify](https://github.com/safishamsi/graphify)** — AI coding assistant skill (`/graphify`) that turns any folder into a queryable knowledge graph. Three passes: AST (code, no LLM), Whisper transcription (audio/video), Claude subagents (docs/PDFs/images). Outputs `graph.html` (interactive), `GRAPH_REPORT.md` (god nodes, communities), `graph.json` (queryable). Graph clustering is topology-based via Leiden — no embeddings needed for community detection. Ships an MCP server (`python -m graphify.serve graph.json`) with `query_graph`, `get_node`, `get_neighbors`, `shortest_path`. Tags relationships as EXTRACTED / INFERRED (with confidence) / AMBIGUOUS.

**[RAG-Anything](https://github.com/HKUDS/RAG-Anything)** — All-in-one multimodal RAG built on LightRAG. Full pipeline: multimodal document parsing → content routing (text / images / tables / equations) → knowledge graph construction → hybrid retrieval (vector + graph traversal). Uses MinerU for high-fidelity PDF parsing. Heavier than most tools here but the most complete end-to-end system.

---

### 🗂️ Workspace Frameworks

**[PARA Workspace](https://github.com/pageel/para-workspace)** — Workspace framework for humans and AI agents based on the PARA method (Projects, Areas, Resources, Archive). Generates a structured directory tree with an `_inbox/` landing zone for dropped files, a `.agents/` folder with governed workflows/rules/skills, and a kernel of hard invariants (Archive is immutable cold storage, no loose files at workspace root, etc.). Bash CLI, MIT license.

**[Beads](https://github.com/gastownhall/beads)** — Distributed graph issue/task tracker for AI agents, powered by Dolt (version-controlled SQL). Replaces markdown plans with a dependency-aware graph. Features hash-based IDs (`bd-a1b2`) to prevent merge collisions, typed graph links (`relates_to`, `duplicates`, `supersedes`, `replies_to`), and a compaction/"memory decay" feature that summarizes old closed tasks to save context window. MCP server available (`pip install beads-mcp`).

---

### 🧠 Persistent Memory Servers

**[mnemory](https://github.com/fpytloun/mnemory)** — Self-hosted MCP server for persistent agent memory across conversations. The key idea: a single unified LLM call handles fact extraction, classification (episodic/semantic/procedural), and deduplication together — emitting explicit ADD/UPDATE/DELETE/SKIP decisions per fact rather than blindly appending. Contradiction resolution works automatically: "I drive a Skoda" followed by "I bought a Tesla" triggers an UPDATE rather than storing both. TTL-based expiry (7d for context, 90d for episodic) with reinforcement: reading a memory bumps its expiry forward, so frequently-recalled facts never silently disappear. Pinned "core memories" always load at session start regardless of recency or TTL. Built-in `fsck` health checker runs three phases: duplicate detection, contradiction scanning, and quality checks (prompt injection patterns, oversized facts). 17 MCP tools including `find_memories` (multi-query expansion + rerank) and `ask_memories` (synthesized NL answer). LoCoMo benchmark: 73.2% using GPT-4o-mini. Apache 2.0.

---

### 🔄 Document Conversion

**[MarkItDown](https://github.com/microsoft/markitdown)** — Microsoft's lightweight Python utility for converting anything to Markdown for LLM consumption. Handles PDF, DOCX, PPTX, XLSX, images (EXIF + OCR), audio (speech transcription), HTML, YouTube URLs, CSV, JSON, XML, ZIP. One CLI command or one Python call. Optional LLM for image descriptions. MIT license, plugin-extensible.

```bash
markitdown report.pdf -o report.md
markitdown diagram.png   # describes the image using LLM vision
```

---


---

### ☁️ Cloud Second Brains

**[Reseek](https://reseek.net)** — Paid cloud "second brain" ($9/mo or $99/yr). Ingests notes, links (metadata extraction), PDFs (page-by-page), images (OCR), YouTube videos, and Twitter/X posts into a single semantic search index with an AI chat interface. MCP tools: `search-library`, `get-library-item`, `save-note`, `save-link` — accessed via personal access tokens. iOS/Android PWA. The closest commercial equivalent to a local second-brain stack, but cloud-only with no graph layer, no entity model, and no plain-file access. Validates the MCP-server-as-primary-dev-interface pattern.

Key gaps versus local tools: no knowledge graph, no entity relationships, no offline use, no git-committable store. Key strength: first-class OCR ingest (screenshots, receipts, handwritten notes → searchable text) and YouTube/Twitter ingestion that most local stacks skip.

## What I'm Building

I'm working on a system that combines all of these layers:

1. **RAW** — every ingested file stored unchanged, hash-prefixed
2. **Organized store** — `store/<project>/entities/` + `store/<project>/docs/` with per-folder INDEX.md (Tolaria conventions)
3. **Graph** — `graph/edges.jsonl` with EXTRACTED/INFERRED/AMBIGUOUS tags + weight scores (Graphify's extraction, Beads' vocabulary)
4. **Vector search** — QMD as the search backend (BM25 + vector + reranking, local-first)

Ingestion: `inbox/` → MarkItDown converts → LLM extracts entities → Graphify indexes graph → QMD embeds → INDEX.md updated.

Retrieval: MCP server with `search()` (QMD), `navigate()` (filesystem), `expand()` (graph), `hybrid()` (all three chained).

---

## What Are You Using?

- Are there tools missing from this list?
- What's your ingestion pipeline?
- How do you handle entity extraction — NER, LLM-assisted, manual?
- Local embeddings (nomic-embed, ollama) or API (OpenAI, Voyage)?
- How do you avoid the context window filling up with stale or redundant knowledge?

Especially curious about what's working at scale (1,000+ notes / documents).
