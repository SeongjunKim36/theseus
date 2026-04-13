# Development Environment Setup Guide

> Follow this guide to set up your environment before starting Phase 1.

---

## Prerequisites

- Python 3.12+ (`python --version`)
- [uv](https://docs.astral.sh/uv/) (Python package manager)
- Docker + Docker Compose
- Git
- (Phase 2) JDK 17+ (for building/running the toy Spring app)

---

## 1. Clone the Repository + Install Dependencies

```bash
git clone https://github.com/SeongjunKim36/theseus.git
cd theseus
uv sync
```

---

## 2. Environment Variables

Copy `.env.template` to `.env` and edit:

```bash
cp .env.template .env
```

```env
# .env.template — Phase 1 required environment variables

# Neo4j
NEO4J_URI=bolt://localhost:7687
NEO4J_USER=neo4j
NEO4J_PASSWORD=theseus-dev

# PostgreSQL
POSTGRES_HOST=localhost
POSTGRES_PORT=5432
POSTGRES_DB=theseus
POSTGRES_USER=postgres
POSTGRES_PASSWORD=theseus-dev
DATABASE_URL=postgresql://postgres:theseus-dev@localhost:5432/theseus

# Project settings
LOG_LEVEL=debug
PROJECT_PATH=projects  # Path where target repositories are cloned for parsing

# Phase 2 additions
OTEL_COLLECTOR_ENDPOINT=http://localhost:4317

# Phase 3 additions
CLAUDE_API_KEY=          # Claude API key (required in Phase 3)
EMBEDDING_MODEL=all-MiniLM-L6-v2  # or codebert-base (to be decided after benchmarking)
```

---

## 3. Start Docker Compose

```bash
docker compose up -d
```

**Verify:**
```bash
# Neo4j Browser
open http://localhost:7474
# Login: neo4j / theseus-dev

# PostgreSQL
psql -h localhost -U postgres -d theseus
# Password: theseus-dev
```

---

## 4. Database Migration

```bash
uv run alembic upgrade head
```

---

## 5. Run Tests

```bash
# All tests
uv run pytest

# Phase-specific tests
uv run pytest tests/test_parser/ -v     # Phase 1: Parser
uv run pytest tests/test_graph/ -v      # Phase 1: Graph
uv run pytest tests/test_runtime/ -v    # Phase 2: Runtime
uv run pytest tests/test_mcp/ -v        # Phase 3: MCP
```

---

## 6. CLI Usage

```bash
# Parse a codebase
uv run theseus parse /path/to/spring-project --project-name my-project

# Graph statistics
uv run theseus stats my-project

# Cypher query
uv run theseus query my-project -q "MATCH (c:Class) RETURN c.name LIMIT 10"

# Drift report (Phase 2+)
uv run theseus drift-report my-project --min-days 14

# MCP server (for Claude Code integration)
uv run theseus mcp

# HTTP server (for web UI + OTel ingestion)
uv run theseus serve --port 8000
```

---

## 7. Claude Code MCP Integration (Phase 3+)

Create `.mcp.json` at the project root:

```json
{
  "mcpServers": {
    "theseus": {
      "command": "uv",
      "args": ["--directory", "/path/to/theseus", "run", "theseus", "mcp"],
      "env": {
        "NEO4J_URI": "bolt://localhost:7687",
        "NEO4J_PASSWORD": "theseus-dev"
      }
    }
  }
}
```

Or use the global configuration (`~/.claude/settings.json`):
```json
{
  "mcpServers": {
    "theseus": {
      "command": "uv",
      "args": ["--directory", "/absolute/path/to/theseus", "run", "theseus", "mcp"]
    }
  }
}
```

---

## 8. OTel Agent Setup (Phase 2+)

Attach the OTel Java Agent to a Spring Boot app:

```bash
# Download the Agent JAR
curl -L -o opentelemetry-javaagent.jar \
  https://github.com/open-telemetry/opentelemetry-java-instrumentation/releases/latest/download/opentelemetry-javaagent.jar

# Run the Spring app (with OTel instrumentation)
java -javaagent:opentelemetry-javaagent.jar \
     -Dotel.service.name=my-spring-app \
     -Dotel.exporter.otlp.endpoint=http://localhost:4317 \
     -Dotel.traces.sampler=parentbased_traceidratio \
     -Dotel.traces.sampler.arg=0.1 \
     -jar my-spring-app.jar
```

---

## Neo4j APOC Plugin Functions

APOC functions used in Phases 1-3:

| Function | Purpose | Phase |
|---|---|---|
| `apoc.periodic.iterate` | Batch write large numbers of nodes | Phase 1 |
| `apoc.create.uuids` | Generate node UUIDs (optional) | Phase 1 |
| `apoc.path.subgraphAll` | Extract N-hop subgraphs | Phase 3 |
| `apoc.text.fuzzyMatch` | Fuzzy matching for natural-language keywords | Phase 3 |

Automatically enabled in Docker Compose:
```yaml
NEO4J_PLUGINS: '["apoc"]'
NEO4J_dbms_security_procedures_unrestricted: "apoc.*"
```

---

## Troubleshooting

| Problem | Solution |
|---|---|
| Neo4j connection failure | Check `docker compose logs neo4j`. The APOC plugin download may still be in progress. |
| PostgreSQL connection failure | Check status with `docker compose ps`. Verify port 5432 is not in use by another process. |
| tree-sitter parsing failure | Verify `tree-sitter-language-pack` version. Python 3.12+ is required. |
| OTel spans not arriving | Check Collector logs: `docker compose logs otel-collector` |
| MCP connection failure | Verify the `.mcp.json` path. Run `uv run theseus mcp` directly to check for errors. |
