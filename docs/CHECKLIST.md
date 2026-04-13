# Theseus — Master Checklist

> Integrated checklist across all Phases. Each gate must be passed before entering the next Phase.

---

## Phase 1: Static Graph MVP (Week 1–4)

### Week 1: Infrastructure + Parser Basics
- [ ] Project initialization (pyproject.toml, uv, ruff, pytest)
- [ ] Docker Compose (Neo4j 5.x + PostgreSQL 16) operational
- [ ] PostgreSQL schema migration (projects, file_hashes, parse_errors)
- [ ] Create toy Spring project (6+ Java, MyBatis XML, application.yml, dead code)
- [ ] tree-sitter Java parser basics (class/method/import/calls)
- [ ] 5+ fixture-based unit tests passing

### Week 2: Spring Analysis + Neo4j
- [ ] tree-sitter edge cases (generics, lambdas, nested classes, enums)
- [ ] Spring annotation analyzer (Controller/Service/Repository/Autowired/Bean)
- [ ] Neo4j schema design + indexes (including Ariadne node reservations)
- [ ] Graph builder (NetworkX → Neo4j batch write)
- [ ] Integration test: toy project → Neo4j → Cypher queries

### Week 3: MyBatis/Config + Incremental Parsing
- [ ] MyBatis XML parser (static SQL + regex table extraction)
- [ ] Config parser (application.yml → DB/API settings)
- [ ] Parsing error recovery (per-file try/except, partial graph guaranteed)
- [ ] Incremental parsing (Git diff + SHA-256 hash cache)
- [ ] PG ↔ Neo4j sync strategy implementation

### Week 4: dongne-v2 + CLI
- [ ] dongne-v2 parsing successful (100+ nodes, 200+ edges)
- [ ] CLI: `theseus parse`, `theseus query`, `theseus stats`
- [ ] 4+ Cypher queries working
- [ ] Parsing accuracy: Precision 90%+, Recall 80%+

### Phase 1 Gate
- [ ] Toy project → Neo4j E2E
- [ ] dongne-v2 → meaningful graph
- [ ] `theseus parse` CLI operational

---

## Phase 2: Runtime Drift Layer (Week 5–8)

### Week 5: OTel Collection
- [ ] Add OTel Collector to Docker Compose
- [ ] Toy app OTel Java Agent → span reception confirmed
- [ ] TraceReceiver: OTLP JSON → ParsedSpan
- [ ] TraceMapper: 4 mapping strategies (code_attr, http_route, sql, external_url)

### Week 6: Drift Detection
- [ ] Neo4j schema extension (runtime_count, last_called_at, source)
- [ ] RuntimeWeightUpdater: span → edge weights
- [ ] DriftDetector: STATIC_ONLY / RUNTIME_ONLY detection
- [ ] Golden dataset validation passing
- [ ] `theseus drift-report` CLI

### Week 7–8: Staging Deployment (parallel with Phase 3)
- [ ] dongne-v2 staging OTel setup
- [ ] Minimum 1 week of trace accumulation
- [ ] 3+ dead code candidates discovered
- [ ] 1+ RUNTIME_ONLY edge discovered
- [ ] dongne-v2 team cross-check

### Phase 2 Gate
- [ ] Toy app OTel → Neo4j weights E2E
- [ ] Golden dataset drift test passing
- [ ] `theseus drift-report` CLI operational

---

## Phase 3: MCP Server + AI Semantic Layer (Week 7–9)

### Week 7: MCP Server Basics
- [ ] FastMCP server skeleton
- [ ] `theseus/get_subgraph` tool
- [ ] `theseus/blast_radius` tool
- [ ] `theseus/drift_report` tool
- [ ] Embedding model benchmark + node embedding generation

### Week 8: Natural Language Queries
- [ ] Natural language → subgraph pipeline (keyword + embedding + graph expansion)
- [ ] `theseus/snapshot` tool
- [ ] Claude API subgraph summarization
- [ ] "Payment-related code" → 5+ related nodes returned

### Week 9: Claude Code Integration
- [ ] Claude Code MCP connection + live queries
- [ ] MCP timeout/error handling
- [ ] Large subgraph truncation
- [ ] E2E: natural language → subgraph → summary

### Phase 3 Gate
- [ ] 4 MCP tools callable from Claude Code (ariadne_bridge is Phase 4)
- [ ] Natural language query → subgraph returned
- [ ] Graceful response on error (no crash)

---

## Phase 4: Ariadne Bridge (Week 10–11)

### Week 10: Bridge Implementation
- [ ] AriadneConnector: 4 mapping rules (DB, API, S3, Cache)
- [ ] Bridge edge creation
- [ ] `theseus/ariadne_bridge` MCP tool

### Week 11: Validation
- [ ] Code → infrastructure end-to-end tracing E2E
- [ ] Reverse query ("What code uses this RDS?")
- [ ] Cytoscape.js displaying code + infrastructure nodes

### Phase 4 Gate
- [ ] Successful code → infrastructure Neo4j traversal
- [ ] MCP bridge query operational

---

## Phase 5: Portfolio Finalization (Week 12–13)

### Week 12: Web Viewer + Documentation
- [ ] Graph viewer (Cytoscape.js)
- [ ] Drift dashboard
- [ ] README complete
- [ ] Setup guides (setup, mcp, otel)

### Week 13: Release
- [ ] Case study blog post
- [ ] AI agent blog post
- [ ] Demo video (2–3 min)
- [ ] GitHub public + Apache-2.0
- [ ] CI/CD (GitHub Actions)
- [ ] Release `v0.1.0`

### Phase 5 Gate (Project Complete)
- [ ] Code graph visualization in web viewer
- [ ] README + demo video
- [ ] GitHub public

---

## Final Success Metrics Verification

| # | Metric | Status |
|---|---|---|
| 1 | "Show me payment-related code" → subgraph returned | [ ] |
| 2 | PR blast radius → CI comment (stretch) | [ ] |
| 3 | Static vs runtime drift detection | [ ] |
| 4 | AI grep 10x → MCP 1x | [ ] |
| 5 | 50% reduction in AI code comprehension tokens | [ ] |
| 6 | Code → infrastructure end-to-end tracing | [ ] |

---

## Pre-Requisite Action Items (Execute immediately in Week 1)

- [ ] **Request dongne-v2 staging access** (needed for Phase 2, approval takes 2–4 weeks)
- [ ] Check Ariadne project progress status (Phase 4 prerequisite)
- [ ] Obtain Claude API key
- [ ] Clean up GitHub repository (currently only specs → initialize project structure)
