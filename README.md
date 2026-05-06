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
**[Tana](https://tana.inc/)** — Outliner-style personal second brain built around paragraph-sized atomic nodes rather than full markdown pages. Each node is a discrete, typed unit — better suited for agent context slicing than page-level retrieval, since LLMs can grab exactly the relevant paragraph without wading through whole documents. Ships an MCP server so agents and humans read and write the same structured graph. Good fit for knowledge that grows incrementally through captured thoughts rather than bulk document ingestion.


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

**[LightRAG](https://github.com/hkuds/lightrag)** — Academic RAG system (EMNLP 2025) combining vector search with a knowledge graph via dual-level retrieval: *local* (specific entities and their immediate neighborhood) and *global* (community/theme patterns across the whole corpus). The engine underlying RAG-Anything. Beats Microsoft GraphRAG on retrieval diversity — 77% vs 23% on multiple domains. The global retrieval level is the key contribution: instead of finding a precise fact, it surfaces recurring themes across all documents by operating at the graph community level. Pluggable storage (local / MongoDB / PostgreSQL / Neo4j / OpenSearch). Ships OpenAI-compatible chat endpoint alongside REST API. Recommends 32B+ model for indexing. MIT license.

**[Fortemi](https://github.com/Fortemi/fortemi)** — Self-hosted semantic knowledge base built in Rust (~160k lines). Hybrid search: BM25 + dense embeddings + Reciprocal Rank Fusion with diversity reranking. Automatic knowledge graph with community detection and recursive exploration. 13 extraction adapters cover 131 document types: images, audio, video, 3D models, emails, and spreadsheets. **43 MCP tools** for agent workflows. PostgreSQL + pgvector + PostGIS backend. Multi-provider LLM support: Ollama, OpenAI, OpenRouter, llama.cpp. Docker Compose deployment or HotM desktop app. Requires 8GB+ GPU VRAM.

---

### 🗂️ Workspace Frameworks

**[PARA Workspace](https://github.com/pageel/para-workspace)** — Workspace framework for humans and AI agents based on the PARA method (Projects, Areas, Resources, Archive). Generates a structured directory tree with an `_inbox/` landing zone for dropped files, a `.agents/` folder with governed workflows/rules/skills, and a kernel of hard invariants (Archive is immutable cold storage, no loose files at workspace root, etc.). Bash CLI, MIT license.

**[Beads](https://github.com/gastownhall/beads)** — Distributed graph issue/task tracker for AI agents, powered by Dolt (version-controlled SQL). Replaces markdown plans with a dependency-aware graph. Features hash-based IDs (`bd-a1b2`) to prevent merge collisions, typed graph links (`relates_to`, `duplicates`, `supersedes`, `replies_to`), and a compaction/"memory decay" feature that summarizes old closed tasks to save context window. MCP server available (`pip install beads-mcp`).

---

### 🧠 Persistent Memory Servers

**[mnemory](https://github.com/fpytloun/mnemory)** — Self-hosted MCP server for persistent agent memory across conversations. The key idea: a single unified LLM call handles fact extraction, classification (episodic/semantic/procedural), and deduplication together — emitting explicit ADD/UPDATE/DELETE/SKIP decisions per fact rather than blindly appending. Contradiction resolution works automatically: "I drive a Skoda" followed by "I bought a Tesla" triggers an UPDATE rather than storing both. TTL-based expiry (7d for context, 90d for episodic) with reinforcement: reading a memory bumps its expiry forward, so frequently-recalled facts never silently disappear. Pinned "core memories" always load at session start regardless of recency or TTL. Built-in `fsck` health checker runs three phases: duplicate detection, contradiction scanning, and quality checks (prompt injection patterns, oversized facts). 17 MCP tools including `find_memories` (multi-query expansion + rerank) and `ask_memories` (synthesized NL answer). LoCoMo benchmark: 73.2% using GPT-4o-mini. Apache 2.0.

**[MemPalace](https://github.com/MemPalace/mempalace)** — Best-benchmarked open-source AI memory system: **96.6% R@5 on LongMemEval with zero LLM calls**. The core insight: for conversation memory, storing verbatim and retrieving by semantic search outperforms summarization and entity extraction. Hybrid v4 (+ BM25 keyword boosting + temporal proximity) reaches 98.4%. Organizes memory as Palace → Wings (people/projects) → Rooms (topics) → Drawers (verbatim content). Temporal entity graph with validity windows — facts expire (`valid_from` / `valid_until`). 29 MCP tools. Auto-save Claude Code hooks (periodic + pre-compression). MIT license.

```bash
mempalace wake-up   # session init: loads recent context + key entities
```

**[OB1 (OpenBrain)](https://github.com/NateBJones-Projects/OB1)** — PostgreSQL-native shared memory protocol. Not an assistant — a shared brain any AI client (Claude, ChatGPT, Cursor, Gemini) can connect to via MCP. Two-level graph: entity-to-entity structural edges and thought-to-thought epistemic edges (`supports`, `contradicts`, `evolved_into`, `supersedes`, `depends_on`). DB trigger auto-queues entity extraction on every insert, skipping unchanged content via SHA256 fingerprint. Edge vocabulary enforced at the DB layer via CHECK constraint — bad LLM labels get DB rejections, not silent bad rows. `support_count` accumulates evidence per edge. Haiku/Opus hybrid classifier caps extraction cost (~67% savings vs Opus-only). Supabase + pgvector backend.

**[Honcho](https://github.com/plastic-labs/honcho)** — Peer-centric memory library for stateful agents. Models both users and AI agents as "Peers" with sessions, message history, and derived representations. Solves user modeling (who is this person, how do they think?) rather than document knowledge storage. Key pattern: **async deriver** — message storage is instant; insight extraction (summaries, peer cards, representations) runs in background. `session.context(tokens=N)` returns the best representation within a token budget — recent messages + older summaries fitting within N tokens. Managed cloud or self-hostable. AGPL-3.0.

**[Cognee](https://github.com/topoteretes/cognee)** — Open-source memory infrastructure for AI agents combining embeddings, knowledge graphs, and cognitive science. Three memory types: permanent graph memory (continuously updated from interactions), session memory (fast cache that syncs to the graph in background), and multi-tenant isolation for enterprise use. Ingests any format, auto-routes queries to optimal search strategy (semantic, graph-based, or hybrid). Ontology grounding, OTEL observability, and audit trails. MCP integration. Cognee Cloud managed service or self-hosted (Modal, Railway, Fly.io, Render). Apache 2.0.

**[memvid](https://github.com/memvid/memvid)** — Single-file memory layer for AI agents: packages data, embeddings, search structures, and metadata into one portable `.mv2` capsule — no database required. Smart Frames design: append-only immutable memory units grouped for efficient compression and parallel reads. BM25 full-text + HNSW vector search. Sub-5ms local memory access. Timeline inspection shows how knowledge evolved; rewind to previous states. Multi-modal: text, images (CLIP), audio (Whisper). Encrypted capsules (`.mv2e`). Fully offline and model-agnostic.

---

### 🔄 Document Conversion

**[MarkItDown](https://github.com/microsoft/markitdown)** — Microsoft's lightweight Python utility for converting anything to Markdown for LLM consumption. Handles PDF, DOCX, PPTX, XLSX, images (EXIF + OCR), audio (speech transcription), HTML, YouTube URLs, CSV, JSON, XML, ZIP. One CLI command or one Python call. Optional LLM for image descriptions. MIT license, plugin-extensible.

```bash
markitdown report.pdf -o report.md
markitdown diagram.png   # describes the image using LLM vision
```

---


---

### 🖥️ Desktop AI Assistants

**[Thoth](https://github.com/siddsachar/Thoth)** — Local-first desktop AI assistant with a personal knowledge graph, 20+ tools, and multi-channel messaging (Telegram, Slack, WhatsApp, Discord, SMS). NiceGUI interface, runs via Ollama or cloud models. Memory stack: SQLite (WAL) → NetworkX graph (in-memory mirror) → FAISS vector index → Obsidian-compatible markdown wiki vault.

Key patterns: **67 valid relation types** with 60+ alias mappings — vague labels like `related_to` and `associated_with` are banned and rejected before saving. **Dream cycle** (nightly 1–5 AM): duplicate merge by semantic similarity, description enrichment for thin entities, confidence decay on stale inferred edges older than 90 days, relationship inference for co-occurring pairs, and insight analysis. **Map-reduce document extraction**: split into 6K-char windows → summarize each → combine → extract up to 12 entities/doc. Hub entity + `extracted_from` provenance edges: every entity points back to its source document. v3.18.0, MIT license.

---

### ☁️ Cloud Second Brains

**[Reseek](https://reseek.net)** — Paid cloud "second brain" ($9/mo or $99/yr). Ingests notes, links (metadata extraction), PDFs (page-by-page), images (OCR), YouTube videos, and Twitter/X posts into a single semantic search index with an AI chat interface. MCP tools: `search-library`, `get-library-item`, `save-note`, `save-link` — accessed via personal access tokens. iOS/Android PWA. The closest commercial equivalent to a local second-brain stack, but cloud-only with no graph layer, no entity model, and no plain-file access. Validates the MCP-server-as-primary-dev-interface pattern.

Key gaps versus local tools: no knowledge graph, no entity relationships, no offline use, no git-committable store. Key strength: first-class OCR ingest (screenshots, receipts, handwritten notes → searchable text) and YouTube/Twitter ingestion that most local stacks skip.
---

### 🔬 Research & Benchmarks

**[Roampal Labs](https://github.com/roampal-ai/roampal-labs)** — Research benchmark, not a product. Tests conversational memory architectures on the corrected LoCoMo dataset (1,986 questions, 10 conversations, 3,015 life facts). End-to-end answer accuracy graded by two independent models — stricter than retrieval recall alone. Key findings:

- **Architecture beats model choice by 10x**: swapping a local 20B for GPT-4o-mini moves accuracy 1.5–2.5 pts; retrieval architecture contributes 23+ pts
- **Atomic fact extraction >> chunking**: +29.2 pts from extracting discrete one-claim units rather than raw chunks
- **Cross-encoder reranking is essential**: +17.8 Hit@1 over cosine similarity
- **Two-lane retrieval** (summaries lane + facts lane, merged and cross-encoder reranked) is the best architecture found
- **Conversation learning beats verbatim ingestion by 23 pts**: 76.6% vs 53.0% answer accuracy
- **Wilson confidence scoring hurts retrieval** — avoid derived confidence scores in ranking

The empirical case that retrieval architecture, not LLM selection, is the primary lever for memory quality. Apache 2.0.


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
