# Neo4j Schema Design

> Theseus + Ariadne Unified Graph Schema

---

## Node Types

### Theseus Nodes (Code)

```cypher
// Class / Interface
(:Class {
    node_id: String,          // MD5(project_id:file_path:class_name)
    project_id: String,
    name: String,             // "BillService"
    fqn: String,              // "com.example.service.BillService"
    package: String,          // "com.example.service"
    file_path: String,        // "src/main/java/com/.../BillService.java"
    type: String,             // "controller" | "service" | "repository" | "component" | "configuration" | "entity" | "unknown"
    annotations: [String],    // ["@Service", "@Transactional"]
    is_interface: Boolean,
    line_start: Integer,
    line_end: Integer
})

// Method
(:Method {
    node_id: String,
    project_id: String,
    name: String,             // "createBill"
    class_name: String,       // "BillService"
    class_fqn: String,        // "com.example.service.BillService" (for runtime mapping)
    class_id: String,         // node_id of the owning Class
    file_path: String,        // "src/main/java/com/.../BillService.java" (for drift/UI)
    signature: String,        // "public Bill createBill(BillRequest req)"
    visibility: String,       // "public" | "private" | "protected" | "package"
    annotations: [String],    // ["@Transactional", "@Override"]
    http_method: String,      // "GET" | "POST" | ... (if Controller)
    http_path: String,        // "/api/bills" (if Controller)
    return_type: String,
    is_static: Boolean,
    line_start: Integer,
    line_end: Integer
})

// DB Table
(:Table {
    node_id: String,
    project_id: String,
    name: String,             // "bill"
    source: String,           // "mybatis" | "jpa" | "runtime"
    operations: [String]      // ["SELECT", "INSERT", "UPDATE"]
})

// External API
(:ExternalAPI {
    node_id: String,
    project_id: String,
    name: String,             // "ThepayAPI"
    url: String,              // "https://api.thepay.co.kr"
    source: String            // "config" | "code" | "runtime"
})

// Configuration
(:Config {
    node_id: String,
    project_id: String,
    key: String,              // "spring.datasource.url"
    value: String,            // "jdbc:mysql://..."
    file: String,             // "application.yml"
    profile: String           // "default" | "prod" | "dev"
})
```

### Ariadne Nodes (Infrastructure) — Reserved

```cypher
// Node types created by Ariadne in Phase 4 (referenced only by Theseus)
(:RDS { identifier, endpoint, engine, vpc_id })
(:ECS { service_name, cluster, task_definition })
(:S3 { bucket_name, region })
(:ElastiCache { cluster_id, engine, endpoint })
(:ALB { dns_name, arn })
(:VPC { vpc_id, cidr_block })
(:SecurityGroup { group_id, name })
```

---

## Edge Types

### Internal Code Edges

```cypher
// Class → Method ownership
(:Class)-[:HAS_METHOD]->(:Method)

// Method → Method call
(:Method)-[:CALLS {
    source: String,           // "static" | "runtime" | "both"
    runtime_count: Integer,   // cumulative call count (default 0)
    last_called_at: DateTime, // last invocation time
    first_seen_at: DateTime,  // first discovery time
    confidence: Float         // mapping confidence (runtime)
}]->(:Method)

// Method → Table query
(:Method)-[:QUERIES {
    sql: String,              // original SQL (abbreviated)
    operation: String,        // "SELECT" | "INSERT" | "UPDATE" | "DELETE"
    source: String,           // "mybatis" | "jpa" | "runtime"
    is_dynamic: Boolean       // whether MyBatis dynamic SQL
}]->(:Table)

// Method → External API call
(:Method)-[:CALLS_API {
    http_method: String,      // "GET" | "POST"
    source: String,           // "static" | "runtime"
    runtime_count: Integer
}]->(:ExternalAPI)

// Class → Class dependency
(:Class)-[:DEPENDS_ON {
    type: String,             // "injection" | "import" | "inheritance" | "implementation"
    field_name: String        // injected field name (for injection type)
}]->(:Class)

// Class → Config reference
(:Class)-[:USES_CONFIG {
    annotation: String        // "@Value" | "@ConfigurationProperties"
}]->(:Config)
```

### Bridge Edges (Phase 4)

```cypher
// Code → Infrastructure bridges
// Table→RDS: enables full-path tracing (Method→QUERIES→Table→HOSTED_ON→RDS)
(:Table)-[:HOSTED_ON { bridge: 'theseus-ariadne', matched_by: String }]->(:RDS)
(:Config)-[:HOSTED_ON { bridge: 'theseus-ariadne', matched_by: String }]->(:RDS)
(:ExternalAPI)-[:SERVED_BY { bridge: 'theseus-ariadne' }]->(:ECS)
(:Config)-[:USES_BUCKET { bridge: 'theseus-ariadne' }]->(:S3)
(:Config)-[:USES_CACHE { bridge: 'theseus-ariadne' }]->(:ElastiCache)
```

> **Note**: The `Table→HOSTED_ON→RDS` mapping extracts the DB name from the Config DB URL
> and links Table nodes belonging to that DB to the corresponding RDS instance. This enables
> full-path tracing: `Method→QUERIES→Table→HOSTED_ON→RDS→BELONGS_TO→VPC`.

---

## Indexes

```cypher
// Required indexes
CREATE INDEX class_lookup IF NOT EXISTS FOR (n:Class) ON (n.node_id, n.project_id);
CREATE INDEX method_lookup IF NOT EXISTS FOR (n:Method) ON (n.node_id, n.project_id);
CREATE INDEX class_name IF NOT EXISTS FOR (n:Class) ON (n.name, n.project_id);
CREATE INDEX method_name IF NOT EXISTS FOR (n:Method) ON (n.name, n.project_id);
CREATE INDEX table_name IF NOT EXISTS FOR (n:Table) ON (n.name, n.project_id);
CREATE INDEX config_key IF NOT EXISTS FOR (n:Config) ON (n.key, n.project_id);

// Full-text search (for natural-language queries)
CREATE FULLTEXT INDEX class_fts IF NOT EXISTS FOR (n:Class) ON EACH [n.name, n.fqn, n.package];
CREATE FULLTEXT INDEX method_fts IF NOT EXISTS FOR (n:Method) ON EACH [n.name, n.class_name, n.signature];

// Embedding vector index (Phase 3)
// Neo4j 5.x vector index
CREATE VECTOR INDEX class_embedding IF NOT EXISTS
FOR (n:Class) ON (n.embedding)
OPTIONS {indexConfig: {`vector.dimensions`: 384, `vector.similarity_function`: 'cosine'}};
```

---

## Key Cypher Query Patterns

```cypher
// 1. Class dependency tree
MATCH path = (c:Class {name: $name, project_id: $pid})-[:DEPENDS_ON*1..3]->(dep)
RETURN path

// 2. Method call chain (with runtime weights)
MATCH path = (m:Method {name: $method})-[:CALLS*1..3]->(callee)
WHERE m.project_id = $pid
RETURN path, [r in relationships(path) | r.runtime_count] AS weights

// 3. Table impact scope
MATCH (m:Method)-[:QUERIES]->(t:Table {name: $table, project_id: $pid})
MATCH (c:Class)-[:HAS_METHOD]->(m)
RETURN c.name, m.name, c.type

// 4. Dead code candidates
MATCH (m:Method {project_id: $pid})
WHERE NOT ()-[:CALLS {source: 'runtime'}]->(m)
  AND NOT ()-[:CALLS {source: 'both'}]->(m)
  AND m.visibility = 'public'
RETURN m.name, m.class_name, m.file_path

// 5. Full-path tracing (code → infrastructure)
MATCH path = (m:Method {name: $method})-[:QUERIES]->(t:Table)
             -[:HOSTED_ON]->(rds:RDS)-[:BELONGS_TO]->(vpc:VPC)
RETURN path

// 6. Embedding similarity search
CALL db.index.vector.queryNodes('class_embedding', 10, $query_vector)
YIELD node, score
RETURN node.name, node.fqn, score
```
