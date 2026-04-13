# Phase 5: Portfolio Finalization — Detailed Plan

> Duration: Week 12–13 (2 weeks) · Prerequisites: Phase 1–4 complete

---

## Goal

Minimum web viewer + 2 blog posts + demo video + open-source release.

---

## Week 12: Web Viewer + Documentation

### 12-1. Minimum Web Viewer

> Details: See [frontend-spec.md](./frontend-spec.md)

**Stack:** React 18 + TypeScript + Cytoscape.js + Tailwind CSS

**Pages:**
- **Graph Viewer** (main): Interactive code topology visualization
- **Drift Dashboard**: Dead code candidates / hidden path listing
- **Project Statistics**: Node/edge counts, parsing status

### 12-2. README

**Required sections:**
1. One-line introduction + screenshot/GIF
2. Why we built this (problem statement)
3. Why not Backstage/Potpie/Sourcegraph
4. Architecture diagram
5. Quick Start (Docker Compose + CLI)
6. MCP connection guide (Claude Code / Cursor)
7. Key MCP tool usage examples
8. Roadmap / Contributing

### 12-3. Setup Guide Documentation

- `docs/setup-guide.md`: Docker Compose + environment variables + CLI
- `docs/mcp-connection-guide.md`: How to connect with Claude Code / Cursor
- `docs/otel-setup-guide.md`: How to attach OTel Agent to a Spring Boot app

---

## Week 13: Blog Posts + Demo + Release

### 13-1. Case Study Blog Post

**Title (draft):** "What happened when we built a code topology map for a legacy Spring Boot project"

**Structure:**
1. Problem: The story of not being able to answer "If I change this, what breaks?" in dongne-v2
2. Solution: What Theseus revealed
   - N dead code instances (screenshots)
   - Hidden AOP paths (before/after comparison)
   - Payment flow visualization
3. Metrics: Parsing accuracy, drift detection results
4. Lessons learned: Why static analysis alone is not enough

### 13-2. AI Coding Agent Blog Post

**Title (draft):** "Give your AI coding agent a code GPS — Context innovation with MCP"

**Structure:**
1. Problem: The inefficiency of AI running grep 10 times to understand code from scratch every time
2. Solution: Providing structured context via the Theseus MCP server
3. Before/After: Token usage comparison for the same task
4. MCP tool usage examples (screenshots)
5. Open-source announcement

### 13-3. Demo Video (2–3 minutes)

**Scenario:**
1. (0:00) Project introduction + problem statement (30s)
2. (0:30) Parse codebase with `theseus parse` CLI (30s)
3. (1:00) Explore code graph in the web viewer (30s)
4. (1:30) Claude Code + MCP: "Show me payment-related code" (40s)
5. (2:10) Drift report — discovering dead code (20s)
6. (2:30) Ariadne bridge — code → infrastructure tracing (20s)
7. (2:50) Wrap-up + GitHub link (10s)

### 13-4. Open-Source Release

- [ ] Switch GitHub repository to public
- [ ] License: Apache-2.0
- [ ] GitHub Actions CI: lint + test
- [ ] Clean up `.env.template` (verify secrets removed)
- [ ] First release tag `v0.1.0`

---

## Checklist

### Week 12
- [ ] Web viewer: Graph visualization (Cytoscape.js)
- [ ] Web viewer: Drift dashboard
- [ ] README complete (including architecture diagram)
- [ ] 3 setup guides (setup, mcp-connection, otel-setup)

### Week 13
- [ ] Case study blog post published
- [ ] AI coding agent blog post published
- [ ] Demo video recorded + uploaded
- [ ] GitHub switched to public + Apache-2.0 license
- [ ] CI/CD setup (GitHub Actions)
- [ ] `.env.template` secrets removal verified
- [ ] Release tag `v0.1.0`

### Phase 5 Gate (Project Complete)
- [ ] **Required**: Code graph visualization in web viewer
- [ ] **Required**: README complete
- [ ] **Required**: Demo video published
- [ ] **Required**: GitHub repository public
- [ ] **Recommended**: 2 blog posts published
