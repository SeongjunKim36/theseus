# Phase 4: Ariadne Bridge — Detailed Plan

> Duration: Week 10–11 (2 weeks) · Prerequisites: Phase 1+3 complete, Ariadne Neo4j graph exists

---

## Goal

Connect Theseus (code graph) and Ariadne (infrastructure graph) via Neo4j edges to enable **end-to-end tracing from code to infrastructure**.

---

## Bridge Mapping Rules

### Automatic Mapping Sources

| Code-Side Information | Extraction Location | Infrastructure-Side Mapping Target |
|---|---|---|
| DB URL (`jdbc:mysql://host:3306/dbname`) | `application.yml` | Ariadne RDS node (endpoint matching) |
| External API URL (`https://api.thepay.co.kr`) | `application.yml` / code constants | Ariadne ECS/ALB node (DNS matching) |
| S3 bucket name | `application.yml` / code constants | Ariadne S3 node |
| Redis URL | `application.yml` | Ariadne ElastiCache node |
| SQS/SNS queue name | `application.yml` | Ariadne SQS/SNS node |

### Mapping Engine

```python
# theseus/bridge/ariadne_connector.py

class AriadneConnector:
    """Theseus ↔ Ariadne Neo4j graph bridge"""

    BRIDGE_RULES = [
        DBBridgeRule(),       # DB URL → RDS
        APIBridgeRule(),      # API URL → ECS/ALB
        S3BridgeRule(),       # S3 bucket → S3
        CacheBridgeRule(),    # Redis URL → ElastiCache
    ]

    def create_bridges(self, project_id: str) -> list[BridgeEdge]:
        """Map Config/ExternalAPI nodes from code graph → infrastructure nodes"""
        edges = []
        for rule in self.BRIDGE_RULES:
            edges.extend(rule.match(self.neo4j, project_id))
        return edges

class DBBridgeRule:
    def match(self, neo4j, project_id) -> list[BridgeEdge]:
        """Match DB URL from application.yml → Ariadne RDS endpoint"""
        return neo4j.run("""
            MATCH (cfg:Config {project_id: $pid})
            WHERE cfg.key CONTAINS 'datasource.url'
            WITH cfg, cfg.value AS url
            // Match against Ariadne RDS node endpoint
            MATCH (rds:RDS)
            WHERE url CONTAINS rds.endpoint
            MERGE (cfg)-[:HOSTED_ON {bridge: 'theseus-ariadne'}]->(rds)
            RETURN cfg.key, rds.identifier
        """, pid=project_id)
```

### Neo4j Bridge Edges

```cypher
// Code → Infrastructure bridge edges
(:Config)-[:HOSTED_ON {bridge: 'theseus-ariadne'}]->(:RDS)
(:ExternalAPI)-[:SERVED_BY {bridge: 'theseus-ariadne'}]->(:ECS)
(:Config)-[:USES_BUCKET {bridge: 'theseus-ariadne'}]->(:S3)
(:Config)-[:USES_CACHE {bridge: 'theseus-ariadne'}]->(:ElastiCache)

// End-to-end tracing query
MATCH path = (m:Method)-[:QUERIES]->(t:Table)-[:HOSTED_ON]->(rds:RDS)
             -[:BELONGS_TO]->(vpc:VPC)
RETURN path
```

---

## MCP Tool: `theseus/ariadne_bridge`

```python
@mcp.tool()
async def ariadne_bridge(
    source: str,                              # Code node name
    direction: str = "code_to_infra",         # code_to_infra | infra_to_code
    include_infra_neighbors: bool = True,     # Include infrastructure-side neighbors
) -> BridgeResponse:
```

**Response example:**
```json
{
  "source": "BillMapper",
  "direction": "code_to_infra",
  "bridges": [
    {
      "code_node": {"name": "BillMapper", "type": "repository", "table": "bill"},
      "infra_node": {"name": "dongne-prod-db", "type": "RDS", "engine": "MySQL 8.0"},
      "relation": "HOSTED_ON"
    }
  ],
  "infra_context": {
    "vpc": "vpc-prod-main",
    "other_services_on_same_db": ["task-api", "task-batch", "task-scheduler"],
    "backup": {"s3_bucket": "dongne-db-backup", "retention": "30d"}
  },
  "impact_note": "The bill table is also used by 3 other services. Schema changes require a full impact assessment."
}
```

---

## Checklist

### Week 10
- [ ] AriadneConnector: 4 bridge rules (DB, API, S3, Cache)
- [ ] Neo4j bridge edge creation (`HOSTED_ON`, `SERVED_BY`, `USES_BUCKET`, `USES_CACHE`)
- [ ] `theseus/ariadne_bridge` MCP tool implementation
- [ ] Unit tests: mock Ariadne nodes → bridge matching

### Week 11
- [ ] E2E: "BillMapper → RDS → VPC → other ECS" end-to-end tracing
- [ ] Reverse direction: "What code depends on this RDS?" query
- [ ] Cytoscape.js prototype displaying code + infrastructure nodes together
- [ ] Mapping failure case handling (different URL formats, missing nodes in Ariadne)

### Phase 4 Gate
- [ ] **Required**: Successful Neo4j traversal from code → infrastructure end-to-end
- [ ] **Required**: Bridge queries available via MCP
- [ ] **Recommended**: Reverse direction queries working
- [ ] **Prerequisite**: Ariadne stores infrastructure graph in Neo4j
