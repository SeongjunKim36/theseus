# REST API + OTLP Receiver Contract

> HTTP interface definitions beyond MCP. Used by the web UI, OTel Collector, and CI/CD.

---

## Server Startup

```bash
# MCP server (for Claude Code, stdio)
theseus mcp

# HTTP server (web UI + OTel ingestion + REST API)
theseus serve --port 8000
```

---

## 1. OTel Trace Ingestion (Phase 2)

### `POST /api/traces`

Sent from OTel Collector via the OTLP HTTP protocol.

**Request:** OTLP JSON (standard format)
```json
{
  "resourceSpans": [
    {
      "resource": {
        "attributes": [
          {"key": "service.name", "value": {"stringValue": "dongne-v2"}}
        ]
      },
      "scopeSpans": [
        {
          "spans": [
            {
              "traceId": "abc123...",
              "spanId": "def456...",
              "parentSpanId": "ghi789...",
              "name": "BillService.createBill",
              "kind": 1,
              "startTimeUnixNano": "1713000000000000000",
              "endTimeUnixNano": "1713000000050000000",
              "attributes": [
                {"key": "code.namespace", "value": {"stringValue": "com.example.service.BillService"}},
                {"key": "code.function", "value": {"stringValue": "createBill"}},
                {"key": "http.route", "value": {"stringValue": "/api/bills"}},
                {"key": "db.statement", "value": {"stringValue": "INSERT INTO bill ..."}}
              ]
            }
          ]
        }
      ]
    }
  ]
}
```

**Response:** `200 OK` (acknowledgment) or `500` (processing failure)

**Processing rules:**
- Upon receipt, convert span -> ParsedSpan -> TraceMapper -> update Neo4j weights
- **Deduplication:** Deduplicate by span_id (ignore re-sent spans)
- **Aggregation:** Raw spans are not stored. Only counts are accumulated on edges (`runtime_count += 1`)
- **Retention:** The Neo4j edge `last_called_at` is always updated to the latest value. `runtime_count` is cumulative.
- **Reset:** Runtime data can be reset via the CLI: `theseus reset-runtime --project <name>`

---

## 2. Project Management API

### `GET /api/projects`

```json
{
  "projects": [
    {
      "id": "uuid-1",
      "name": "dongne-v2",
      "repo_path": "/path/to/dongne-v2",
      "status": "ready",
      "last_parsed_at": "2026-04-20T15:30:00Z",
      "stats": {
        "classes": 142,
        "methods": 890,
        "tables": 35,
        "external_apis": 8,
        "static_edges": 450,
        "runtime_edges": 520
      }
    }
  ]
}
```

### `POST /api/projects/{id}/parse`

Trigger parsing (asynchronous).

**Request:** `{"incremental": true}` (default) or `{"incremental": false}` (full scan)
**Response:** `202 Accepted` + `{"task_id": "...", "status": "parsing"}`

### `GET /api/projects/{id}/stats`

```json
{
  "project": "dongne-v2",
  "status": "ready",
  "last_parsed_at": "2026-04-20T15:30:00Z",
  "graph": {
    "classes": 142, "methods": 890, "tables": 35,
    "external_apis": 8, "configs": 23,
    "static_edges": 450, "runtime_edges": 520
  },
  "parsing": {
    "total_files": 152, "success": 148, "failed": 4,
    "success_rate": 0.974,
    "last_incremental": {"files": 12, "duration_ms": 300}
  },
  "layer_distribution": {
    "controller": 12, "service": 38, "repository": 24,
    "entity": 45, "other": 23
  }
}
```

---

## 3. Graph API (for Web UI)

### `GET /api/projects/{id}/graph`

**Parameters:**
- `query` (optional): Filter by natural language or node name
- `depth` (optional, default 2): Graph traversal depth
- `include_runtime` (optional, default true): Include runtime weights

**Response (Cytoscape.js format):**
```json
{
  "elements": {
    "nodes": [
      {
        "data": {
          "id": "abc123",
          "name": "BillService",
          "type": "service",
          "file_path": "src/main/java/com/.../BillService.java",
          "runtime_calls": 24000,
          "drift_type": null,
          "annotations": ["@Service", "@Transactional"]
        }
      }
    ],
    "edges": [
      {
        "data": {
          "id": "edge-1",
          "source": "abc123",
          "target": "def456",
          "type": "DEPENDS_ON",
          "source_type": "both",
          "runtime_count": 24000
        }
      }
    ]
  },
  "metadata": {
    "total_nodes": 8,
    "total_edges": 12,
    "query_time_ms": 120,
    "truncated": false
  }
}
```

> **Note:** The MCP `get_subgraph` response uses its own format (nodes/edges/drift/summary),
> while the REST `/graph` endpoint uses Cytoscape.js format (`elements.nodes/edges`).
> The core logic returns an internal format, and the MCP adapter and REST adapter each handle the conversion.

### `GET /api/projects/{id}/graph/{nodeId}`

Returns node details + 1-hop neighbors.

```json
{
  "node": {
    "id": "abc123",
    "name": "BillService",
    "type": "service",
    "file_path": "src/main/java/com/.../BillService.java",
    "package": "com.example.service",
    "annotations": ["@Service", "@Transactional"],
    "methods": [
      {"name": "createBill", "runtime_calls_30d": 24000, "drift_type": null},
      {"name": "getBill", "runtime_calls_30d": 5000, "drift_type": null},
      {"name": "legacyRefund", "runtime_calls_30d": 0, "drift_type": "STATIC_ONLY"}
    ]
  },
  "neighbors": {
    "upstream": [{"id": "...", "name": "BillController", "type": "controller"}],
    "downstream": [{"id": "...", "name": "BillMapper", "type": "repository"}]
  }
}
```

---

## 4. Drift API (for Web UI)

### `GET /api/projects/{id}/drift`

**Parameters:**
- `min_days` (optional, default 14): Minimum inactive period
- `limit` (optional, default 50): Maximum number of results

**Response:**
```json
{
  "dead_code_candidates": [
    {
      "method": "BillService.legacyRefund",
      "file_path": "src/.../BillService.java",
      "line": 145,
      "days_inactive": 45,
      "static_callers": 1,
      "severity": "medium"
    }
  ],
  "hidden_paths": [
    {
      "caller": "TransactionInterceptor",
      "callee": "BillService.createBill",
      "mechanism": "AOP_PROXY",
      "runtime_count": 24000
    }
  ],
  "summary": {
    "dead_code_count": 3,
    "hidden_path_count": 2,
    "coverage_ratio": 0.87
  }
}
```

---

## MCP vs REST Response Conversion Summary

| Data | MCP Format | REST Format |
|---|---|---|
| Subgraph | `{nodes, edges, drift, summary}` | `{elements: {nodes, edges}, metadata}` (Cytoscape) |
| Node details | MCP `get_subgraph(depth=1)` | `GET /graph/{nodeId}` (includes neighbors) |
| Drift | `{dead_code_candidates, hidden_paths, summary}` | Same |
| Stats | `theseus/snapshot` MCP tool | `GET /stats` |
