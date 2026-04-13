# Phase 3: MCP Server + AI Semantic Layer — Detailed Specification

> Duration: Weeks 7–9 (3 weeks, overlapping with Phase 2 W7–8) · Prerequisite: Phase 1 complete, DEVELOPMENT_PLAN.md §3 Phase 3

---

## Goal

Implement a server that provides structured code topology
to AI coding agents (Claude Code, Cursor, etc.) via the MCP protocol.

---

## MCP Tool Detailed Specs

### Tool 1: `theseus/get_subgraph`

**Purpose:** Return a relevant subgraph from the codebase

```python
@mcp.tool()
async def get_subgraph(
    project: str,                   # Project name (multi-project support, Codex correction)
    query: str,                     # Natural language or node name
    depth: int = 2,                 # Graph traversal depth (1~5, default 2)
    include_runtime: bool = True,   # Include runtime weights
    max_nodes: int = 50,            # Maximum number of returned nodes
) -> SubgraphResponse:
```

**Request example:**
```json
{"query": "billing", "depth": 2, "include_runtime": true, "max_nodes": 30}
```

**Response schema:**
```json
{
  "query": "billing",
  "match_strategy": "embedding_similarity",  // keyword | embedding_similarity | exact
  "nodes": [
    {
      "id": "abc123",
      "name": "BillService",
      "type": "service",
      "file_path": "src/main/java/com/example/service/BillService.java",
      "package": "com.example.service",
      "annotations": ["@Service", "@Transactional"],
      "methods": ["createBill", "getBill", "legacyRefund"],
      "runtime": {
        "total_calls_30d": 24000,
        "last_called_at": "2026-04-12T15:30:00Z"
      }
    },
    {
      "id": "def456",
      "name": "BillMapper",
      "type": "repository",
      "file_path": "src/main/java/com/example/repository/BillMapper.java",
      "tables": ["bill", "bill_detail"]
    }
  ],
  "edges": [
    {
      "from": "abc123",
      "to": "def456",
      "type": "DEPENDS_ON",
      "subtype": "injection",
      "runtime_count": 24000,
      "source": "both"
    }
  ],
  "drift": [
    {
      "node": "BillService.legacyRefund",
      "type": "STATIC_ONLY",
      "days_inactive": 45,
      "severity": "medium"
    }
  ],
  "summary": "Billing flow: BillController -> BillService -> BillMapper (bill table). External payments processed via ThepayClient.",
  "metadata": {
    "total_nodes_in_graph": 342,
    "returned_nodes": 8,
    "query_time_ms": 120
  }
}
```

### Tool 2: `theseus/blast_radius`

**Purpose:** Analyze the scope of code affected by a change

```python
@mcp.tool()
async def blast_radius(
    target: str,                          # File path or "Class.method"
    direction: str = "downstream",        # downstream | upstream | both
    max_depth: int = 3,                   # Impact tracing depth
    include_tables: bool = True,          # Include table impact
) -> BlastRadiusResponse:
```

**Response schema:**
```json
{
  "target": "BillService.createBill",
  "direction": "downstream",
  "impact": {
    "direct": [
      {"name": "BillMapper.insertBill", "type": "repository", "relation": "CALLS"},
      {"name": "NotificationService.sendBillNotice", "type": "service", "relation": "CALLS"}
    ],
    "indirect": [
      {"name": "FirebaseClient.pushNotification", "type": "external_api", "depth": 2}
    ],
    "tables": [
      {"name": "bill", "operations": ["INSERT"], "via": "BillMapper.insertBill"},
      {"name": "notification_log", "operations": ["INSERT"], "via": "NotificationService"}
    ]
  },
  "severity": "high",
  "reason": "Direct modification of billing table + cascading notification service calls. 3 services and 2 tables affected.",
  "total_affected": 5,
  "recommendation": "Verify unit tests for BillMapper and NotificationService"
}
```

### Tool 3: `theseus/drift_report`

**Purpose:** Report static vs. runtime discrepancies

```python
@mcp.tool()
async def drift_report(
    scope: str = "all",       # all | package name | class name
    min_days: int = 14,       # Minimum inactive period
    limit: int = 20,          # Maximum number of results
) -> DriftResponse:
```

**Response schema:**
```json
{
  "scope": "all",
  "period_days": 14,
  "dead_code_candidates": [
    {
      "method": "BillService.legacyRefund",
      "file": "src/.../BillService.java",
      "line": 145,
      "days_inactive": 45,
      "static_callers": 1,
      "severity": "medium",
      "note": "Statically called from BillController, but 0 runtime calls in the last 30 days"
    }
  ],
  "hidden_paths": [
    {
      "caller": "TransactionInterceptor",
      "callee": "BillService.createBill",
      "mechanism": "AOP_PROXY",
      "runtime_count": 24000,
      "note": "@Transactional proxy path not visible in static analysis"
    }
  ],
  "summary": {
    "dead_code_candidates": 3,
    "hidden_paths": 2,
    "total_static_edges": 450,
    "total_runtime_edges": 520,
    "coverage_ratio": 0.87
  }
}
```

### Tool 4: `theseus/snapshot`

**Purpose:** Compressed topology for AI context injection

```python
@mcp.tool()
async def snapshot(
    scope: str = "all",               # all | package name | service name
    format: str = "compact",          # compact | detailed | tree
    max_tokens: int = 2000,           # Response token limit
) -> SnapshotResponse:
```

**Compact format response:**
```json
{
  "project": "dongne-v2",
  "scope": "all",
  "stats": {"classes": 142, "methods": 890, "tables": 35, "external_apis": 8},
  "layers": {
    "controllers": ["BillController", "UserController", "CampusController", "..."],
    "services": ["BillService", "UserService", "NotificationService", "..."],
    "repositories": ["BillMapper", "UserMapper", "CampusMapper", "..."]
  },
  "hot_paths": [
    "BillController -> BillService -> BillMapper -> bill (24k calls/30d)",
    "UserController -> UserService -> UserMapper -> user (18k calls/30d)"
  ],
  "drift_summary": "3 dead code candidates, 2 hidden AOP paths",
  "key_dependencies": [
    "ThepayClient (external payment API)",
    "FirebaseClient (push notifications)",
    "S3Client (file upload)"
  ]
}
```

### Tool 5: `theseus/ariadne_bridge` (to be implemented in Phase 4)

```python
@mcp.tool()
async def ariadne_bridge(
    source: str,              # Code node name (e.g., "BillMapper")
    direction: str = "code_to_infra",  # code_to_infra | infra_to_code
) -> BridgeResponse:
```

---

## AI Semantic Layer Details

### Embedding Model Benchmark (Week 7)

**Candidates:**

| Model | Dimensions | Characteristics | Local | Cost |
|---|---|---|---|---|
| `all-MiniLM-L6-v2` | 384 | General-purpose NLP | Yes | Free |
| `codebert-base` | 768 | Code-specialized | Yes | Free |
| `text-embedding-3-small` | 1536 | OpenAI, high quality | No | Paid |

**Benchmark method:**
```python
# tests/test_intelligence/test_embedding_benchmark.py

QUERIES = [
    ("billing", ["BillService", "BillMapper", "BillController"]),
    ("notification", ["NotificationService", "FirebaseClient"]),
    ("user authentication", ["AuthService", "UserService"]),
]

def benchmark_model(model_name, queries):
    """Search top-5 nodes per query -> measure expected node inclusion rate"""
    model = SentenceTransformer(model_name)
    hits = 0
    total = 0
    for query, expected in queries:
        query_vec = model.encode(query)
        results = search_similar(query_vec, top_k=5)
        for exp in expected:
            total += 1
            if exp in [r.name for r in results]:
                hits += 1
    return hits / total  # Hit Rate
```

### Natural Language -> Subgraph Pipeline

```
User query: "billing-related code"
        |
        v
[Step 1: Keyword Extraction]
  LLM extracts "billing" -> ["bill", "payment", "pay", "billing"]
        |
        v
[Step 2: Node Search]
  (a) Embedding similarity: query_vec -> top-10 similar nodes
  (b) Keyword matching: Neo4j CONTAINS search
  (c) Union + deduplication
        |
        v
[Step 3: Graph Expansion]
  Explore neighbors up to depth N from seed nodes
  Prioritize top edges by runtime weight
        |
        v
[Step 4: Summary Generation]
  Claude API generates natural language description from subgraph
        |
        v
  Final response: nodes + edges + drift + summary
```

### Subgraph Summarization (Claude API)

```python
# theseus/intelligence/summarizer.py

SUMMARY_PROMPT = """
Below is a subgraph from the codebase. Please summarize this code structure in 2-3 sentences.

Nodes: {nodes}
Edges: {edges}
Runtime data: {runtime_info}

When summarizing:
1. What functionality this code is responsible for
2. The main call flow (in A -> B -> C form)
3. Any external dependencies, if present
"""

async def summarize_subgraph(self, nodes, edges, runtime_info) -> str:
    response = await self.claude.messages.create(
        model="claude-sonnet-4-6",
        max_tokens=300,
        messages=[{"role": "user", "content": SUMMARY_PROMPT.format(...)}]
    )
    return response.content[0].text
```

---

## MCP Server Implementation Details

### Server Setup

```python
# theseus/mcp/server.py

from mcp.server.fastmcp import FastMCP

mcp = FastMCP("theseus", description="Code topology MCP server")

# Tools are registered via decorators in tools.py
from theseus.mcp.tools import *  # noqa

def run_mcp():
    """Run MCP server in stdio mode (for Claude Code connection)"""
    mcp.run(transport="stdio")

# Codex correction: HTTP server also needed in addition to MCP (stdio)
# - OTel Collector -> POST /api/traces (trace ingestion)
# - Web UI -> GET /api/projects/* (REST API)
# The same core logic is shared by MCP tools and REST endpoints

from fastapi import FastAPI
api = FastAPI(title="Theseus API")

@api.post("/api/traces")
async def receive_traces(data: dict):
    """Receive OTLP traces from OTel Collector"""
    ...

@api.get("/api/projects/{project_id}/graph")
async def get_graph(project_id: str, query: str = None):
    """Graph API for web UI (converts and returns in Cytoscape format)"""
    ...
```

### Claude Code Connection Configuration

```json
// ~/.claude/settings.json (or project .mcp.json)
{
  "mcpServers": {
    "theseus": {
      "command": "uv",
      "args": ["--directory", "/path/to/theseus", "run", "python", "-m", "theseus.mcp.server"],
      "env": {
        "NEO4J_URI": "bolt://localhost:7687",
        "NEO4J_PASSWORD": "theseus-dev"
      }
    }
  }
}
```

### Error Handling

```python
# theseus/mcp/tools.py

@mcp.tool()
async def get_subgraph(query: str, depth: int = 2, ...) -> dict:
    try:
        # Validate depth range
        depth = max(1, min(depth, 5))
        max_nodes = max(1, min(max_nodes, 100))

        result = await query_engine.search(query, depth, max_nodes)

        if not result.nodes:
            return {"error": None, "nodes": [], "message": f"No results for '{query}'"}

        return result.to_dict()

    except Neo4jServiceUnavailable:
        return {"error": "neo4j_unavailable", "message": "Cannot connect to Neo4j server. Check docker compose up"}
    except TimeoutError:
        return {"error": "timeout", "message": "Query timed out (10s). Try reducing depth or max_nodes"}
    except Exception as e:
        logger.error(f"MCP tool error: {e}")
        return {"error": "internal", "message": str(e)}
```

### Response Size Management

```python
MAX_RESPONSE_TOKENS = 4000  # Maximum tokens for MCP response

def truncate_response(response: dict, max_tokens: int = MAX_RESPONSE_TOKENS) -> dict:
    """Reduce nodes/edges when response exceeds token limit"""
    estimated_tokens = len(json.dumps(response)) // 4  # Rough token estimate

    if estimated_tokens <= max_tokens:
        return response

    # Keep only top nodes by runtime call frequency
    nodes = sorted(response["nodes"],
                   key=lambda n: n.get("runtime", {}).get("total_calls_30d", 0),
                   reverse=True)

    while estimated_tokens > max_tokens and len(nodes) > 5:
        nodes = nodes[:-1]
        # Filter edges to match remaining nodes
        node_ids = {n["id"] for n in nodes}
        edges = [e for e in response["edges"]
                 if e["from"] in node_ids and e["to"] in node_ids]
        response["nodes"] = nodes
        response["edges"] = edges
        estimated_tokens = len(json.dumps(response)) // 4

    response["metadata"]["truncated"] = True
    return response
```

---

## Checklist

### Week 7
- [ ] FastMCP server skeleton (`mcp/server.py`)
- [ ] `theseus/get_subgraph` tool (exact name matching mode)
- [ ] `theseus/blast_radius` tool
- [ ] `theseus/drift_report` tool
- [ ] Embedding model benchmark (`all-MiniLM-L6-v2` vs `codebert-base`)
- [ ] Generate node embeddings with selected model -> store in Neo4j

### Week 8
- [ ] Natural language -> subgraph pipeline (keyword extraction + embedding similarity + graph expansion)
- [ ] `theseus/snapshot` tool
- [ ] `get_subgraph` natural language mode
- [ ] Claude API subgraph summarization
- [ ] Natural language query "billing-related code" -> return 5+ related nodes

### Week 9
- [ ] Claude Code MCP connection setup + live queries
- [ ] MCP timeout handling (10 seconds)
- [ ] Graceful error response when Neo4j is down
- [ ] Large subgraph truncation (token limit)
- [ ] E2E scenario: "Show me billing-related code" -> subgraph -> summary
- [ ] Response token usage measurement + optimization

### Phase 3 Gate
- [ ] **Required**: All 5 MCP tools callable from Claude Code
- [ ] **Required**: Natural language query -> return relevant subgraph
- [ ] **Required**: Return error response without crashing on failure
- [ ] **Recommended**: Deliver meaningful information within 2000 response tokens
- [ ] **Recommended**: Embedding similarity search Hit Rate 70%+
