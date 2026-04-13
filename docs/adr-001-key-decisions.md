# ADR-001: Key Technical Decision Records

> Architecture Decision Records · 2026-04-13

---

## Decision 1: Potpie AI — Reference, Not Fork

**Status:** Accepted (2026-04-13)

**Context:** Potpie AI is an open-source project with 5300+ stars that already implements an AST→Neo4j pipeline. Forking would be fast, but 80% of it consists of SaaS dependencies (Firebase, Stripe, Celery, Redis, LangChain).

**Decision:** Use as reference (study patterns only) and implement from scratch.

**Reference targets:**
- `app/modules/parsing/graph_construction/parsing_repomap.py` — RepoMap graph build pattern
- `app/modules/parsing/graph_construction/queries/tree-sitter-java-tags.scm` — Java tag queries
- `app/modules/parsing/graph_construction/code_graph_service.py` — Neo4j batch writes (UNWIND in batches of 1000)
- `app/modules/intelligence/agents/chat_agents/system_agents/blast_radius_agent.py` — BlastRadius prompt
- `app/modules/parsing/utils/content_hash.py` — File-hash-based incremental parsing
- `compose.yaml` — PostgreSQL + Neo4j + Redis composition

**Outcome:** Minimized dependencies, eliminated maintenance burden. Core parsing logic is ~500 lines, so the cost of a from-scratch implementation is low.

---

## Decision 2: MCP Server Language — Python

**Status:** Accepted (2026-04-13)

**Context:** The MCP SDK supports both Python and TypeScript. Potpie reference code is Python, and the tree-sitter ecosystem is stronger in Python.

**Decision:** Python (`fastmcp` SDK)

**Rationale:**
- Same language as the Potpie reference code, making pattern adoption easier
- tree-sitter-language-pack is Python-native
- FastAPI and the MCP server can coexist in the same process
- The Neo4j Python driver is mature

---

## Decision 3: Embedding Model — To Be Decided After Benchmarking (Phase 3 Week 7)

**Status:** Pending (benchmark scheduled)

**Candidates:**
| Model | Dimensions | Characteristics |
|---|---|---|
| `all-MiniLM-L6-v2` | 384 | General-purpose NLP, local, free |
| `codebert-base` | 768 | Code-specialized, local, free |

**Benchmark criteria:** 10 natural-language queries → top-5 node retrieval Hit Rate
**Decision target:** Phase 3 Week 7

---

## Decision 4: Phase 2 Instrumentation Scope — Start with a Specific dongne-v2 Module

**Status:** Accepted (2026-04-13)

**Decision:** Instrument the billing module (`bill` package) first, then expand to the full codebase after success.

**Rationale:**
- Full instrumentation risks excessive trace volume and uncertain performance impact
- The billing module directly relates to success metrics 1 and 4
- Start with 10% sampling rate

---

## Decision 5: Server Architecture — MCP (stdio) + FastAPI (HTTP) Side-by-Side

**Status:** Accepted (2026-04-13, incorporating Codex review feedback)

**Context:** MCP connects to Claude Code via stdio transport. However, we also need OTel Collector trace ingestion and a REST API for the web UI.

**Decision:** MCP tools and FastAPI endpoints share the same core logic.

```
theseus/
├── core/           # Shared business logic (graph queries, drift, summaries)
├── mcp/server.py   # MCP tool → core calls (stdio)
└── api/router.py   # REST endpoint → core calls (HTTP)
```

**Execution:**
- `theseus mcp` → stdio MCP server (for Claude Code)
- `theseus serve` → FastAPI HTTP server (for web UI + OTel ingestion)

---

## Decision 6: node_id Generation Rules

**Status:** Accepted (2026-04-13)

**Rule:** `MD5(project_id + ":" + file_path + ":" + identifier)`

| Node Type | identifier Example | node_id Example |
|---|---|---|
| Class | `com.example.service.BillService` | `MD5("proj1:src/.../BillService.java:com.example.service.BillService")` |
| Method | `com.example.service.BillService.createBill` | `MD5("proj1:src/.../BillService.java:...BillService.createBill")` |
| Table | `bill` | `MD5("proj1:mybatis:bill")` |
| ExternalAPI | `https://api.thepay.co.kr` | `MD5("proj1:api:https://api.thepay.co.kr")` |
| Config | `spring.datasource.url` | `MD5("proj1:config:spring.datasource.url")` |

**Rationale:** Same pattern as Potpie. MD5 has an extremely low collision risk and produces a 32-character hex string well-suited for Neo4j indexing.
