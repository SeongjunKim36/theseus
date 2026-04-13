# Phase 1: Static Graph MVP — Detailed Specification

> Duration: Week 1–4 (4 weeks) · Prerequisite: DEVELOPMENT_PLAN.md §3 Phase 1

---

## Goal

Parse a Java/Spring Boot codebase with tree-sitter to generate a Neo4j code graph.
Validate on a toy project first, then extend to dongne-v2.

---

## Week 1: Infrastructure + tree-sitter Base Parser

### 1-1. Project Initialization

**Tasks:**
- Create `pyproject.toml` (uv package manager)
- Python 3.12+, dependencies: `tree-sitter`, `tree-sitter-language-pack`, `neo4j`, `networkx`, `pyyaml`, `click`, `fastapi`, `sqlalchemy`, `alembic`
- Configure ruff (linter/formatter), pytest
- `.gitignore`, `.env.template`

**Completion Criteria:**
```bash
uv sync          # Dependencies installed successfully
uv run pytest    # Empty tests pass
uv run ruff check .  # Lint passes
```

### 1-2. Docker Compose

**Tasks:**
```yaml
services:
  neo4j:
    image: neo4j:5-community
    environment:
      NEO4J_AUTH: neo4j/theseus-dev
      NEO4J_PLUGINS: '["apoc"]'
    ports: ["7474:7474", "7687:7687"]
    volumes: [neo4j_data:/data]

  postgres:
    image: postgres:16
    environment:
      POSTGRES_DB: theseus
      POSTGRES_PASSWORD: theseus-dev
    ports: ["5432:5432"]
    volumes: [postgres_data:/var/lib/postgresql/data]
```

**Completion Criteria:**
```bash
docker compose up -d
# Neo4j Browser: verify access at http://localhost:7474
# PostgreSQL: verify access via psql -h localhost -U postgres -d theseus
```

### 1-3. PostgreSQL Metadata Schema

**Tasks:**
- Initialize Alembic + create initial migration
- Table design:

```sql
-- Project management
CREATE TABLE projects (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    repo_path TEXT NOT NULL,
    last_parsed_at TIMESTAMP,
    status VARCHAR(50) DEFAULT 'pending',  -- pending | parsing | ready | error
    created_at TIMESTAMP DEFAULT now()
);

-- File hash cache (for incremental parsing)
CREATE TABLE file_hashes (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id UUID REFERENCES projects(id) ON DELETE CASCADE,
    file_path TEXT NOT NULL,
    content_hash VARCHAR(64) NOT NULL,  -- SHA-256
    last_parsed_at TIMESTAMP DEFAULT now(),
    UNIQUE(project_id, file_path)
);

-- Parse error log
CREATE TABLE parse_errors (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id UUID REFERENCES projects(id) ON DELETE CASCADE,
    file_path TEXT NOT NULL,
    error_type VARCHAR(100),
    error_message TEXT,
    created_at TIMESTAMP DEFAULT now()
);
```

**Completion Criteria:**
```bash
uv run alembic upgrade head   # Migration succeeds
# Verify 3 tables exist in psql
```

### 1-4. Toy Spring Project

**Tasks:**
Create a minimal Spring Boot project in `tests/fixtures/sample-spring-app/`:

```
sample-spring-app/
├── src/main/java/com/example/
│   ├── controller/
│   │   └── BillController.java       # @RestController, REST endpoints
│   ├── service/
│   │   ├── BillService.java          # @Service, business logic
│   │   └── NotificationService.java  # @Service, called from BillService
│   ├── repository/
│   │   └── BillMapper.java           # MyBatis @Mapper interface
│   ├── model/
│   │   └── Bill.java                 # Entity
│   └── config/
│       └── AppConfig.java            # @Configuration, @Bean
├── src/main/resources/
│   ├── application.yml               # Includes DB URL, external API URL
│   └── mapper/
│       └── BillMapper.xml            # MyBatis SQL (static + dynamic)
└── build.gradle
```

**Patterns to include:**
- `@RestController` -> `@Service` -> `@Mapper` call chain
- `@Autowired` / constructor injection
- `@Transactional` (to be supplemented with Phase 2 runtime data)
- MyBatis XML: static SQL + `<if>`, `<foreach>` dynamic SQL
- `application.yml`: DB URL, external payment API URL
- Intentional dead code: `BillService.legacyRefund()` (a method that is never called)

**Completion Criteria:**
- 6+ Java files
- 1+ MyBatis XML
- 1 application.yml
- 1+ intentional dead code method

### 1-5. tree-sitter Java Parser (Basic)

**Tasks:**
```python
# theseus/parser/tree_sitter_parser.py

class JavaParser:
    """tree-sitter based Java source file parser"""

    def parse_file(self, file_path: str) -> ParseResult:
        """Parse a single Java file -> extract nodes/edges"""
        ...

    def parse_directory(self, dir_path: str) -> list[ParseResult]:
        """Parse all Java files in a directory"""
        ...

@dataclass
class ParseResult:
    file_path: str
    classes: list[ClassNode]
    methods: list[MethodNode]
    imports: list[str]
    calls: list[CallEdge]        # Method -> method calls
    errors: list[ParseError]     # Parse failure info
```

**tree-sitter Queries (based on Potpie java-tags.scm):**
```scheme
;; theseus/parser/queries/java-tags.scm
;; Class declarations
(class_declaration name: (identifier) @class.name) @class.def

;; Method declarations
(method_declaration name: (identifier) @method.name) @method.def

;; Method invocations
(method_invocation name: (identifier) @call.name) @call.ref

;; Imports
(import_declaration (scoped_identifier) @import.name)

;; Annotations
(annotation name: (identifier) @annotation.name)
```

**Scope for W1:**
- Class/interface declarations
- Method declarations + signatures
- Import statements
- Method calls (simple cases)

**Out of scope for W1 (deferred to W2):**
- Generics (`List<Map<K,V>>`)
- Lambdas/method references
- Nested classes / anonymous classes
- Enums

**Completion Criteria:**
```python
# Parse BillController.java from the toy project
result = parser.parse_file("BillController.java")
assert len(result.classes) == 1
assert result.classes[0].name == "BillController"
assert len(result.methods) >= 2  # REST endpoints
assert len(result.calls) >= 1    # BillService call
```

### 1-6. Fixture-Based Unit Tests

**Tasks:**
```python
# tests/test_parser/test_tree_sitter_parser.py

class TestJavaParser:
    @pytest.fixture
    def parser(self):
        return JavaParser()

    @pytest.fixture
    def sample_app_path(self):
        return Path("tests/fixtures/sample-spring-app/src/main/java")

    def test_parse_controller(self, parser, sample_app_path):
        """Parse BillController -> verify class, method, and call extraction"""
        result = parser.parse_file(sample_app_path / "com/example/controller/BillController.java")
        assert result.classes[0].name == "BillController"
        # ...

    def test_parse_service_calls(self, parser, sample_app_path):
        """Verify BillService -> NotificationService call relationship extraction"""
        ...

    def test_parse_error_resilience(self, parser, tmp_path):
        """Syntax error file -> returns ParseError, no crash"""
        broken = tmp_path / "Broken.java"
        broken.write_text("public class { syntax error")
        result = parser.parse_file(broken)
        assert len(result.errors) > 0
```

**Completion Criteria:**
- 5+ tests written
- `uv run pytest tests/test_parser/ -v` all pass

---

## Week 2: tree-sitter Stabilization + Spring Annotations

### 2-1. tree-sitter Edge Case Handling

**Cases to handle:**

| Case | Example | Approach |
|---|---|---|
| Generics | `List<Map<String, Object>>` | Handle tree-sitter `type_arguments` node |
| Lambdas | `list.stream().map(x -> x.getName())` | Extract calls inside `lambda_expression` |
| Nested classes | `class Outer { class Inner {} }` | Use parent reference for `Outer.Inner` form |
| Enums | `enum Status { ACTIVE, INACTIVE }` | Separate `enum_declaration` handling |
| Static imports | `import static org.util.Assert.*` | Static flag |
| Method references | `this::processItem` | `method_reference` node |

**Completion Criteria:**
- Test fixtures + unit tests passing for each case
- Sample 10 Java files from dongne-v2 -> parse successfully

### 2-2. Spring Annotation Analyzer

**Tasks:**
```python
# theseus/parser/spring_analyzer.py

class SpringAnalyzer:
    """Spring annotation-based bean/layer analysis"""

    LAYER_ANNOTATIONS = {
        "RestController": "controller",
        "Controller": "controller",
        "Service": "service",
        "Repository": "repository",
        "Component": "component",
        "Configuration": "configuration",
    }

    INJECTION_ANNOTATIONS = {"Autowired", "Inject", "Value"}

    def analyze(self, parse_result: ParseResult) -> SpringMetadata:
        """Extract Spring metadata from parse results"""
        ...

@dataclass
class SpringMetadata:
    bean_type: str | None          # controller, service, repository, ...
    injected_dependencies: list[str]  # Bean class names injected
    request_mappings: list[RequestMapping]  # HTTP endpoints
    transactional_methods: list[str]  # @Transactional methods (reference for Phase 2)
    event_listeners: list[str]       # @EventListener methods
```

**Annotations to recognize:**

| Annotation | Extracted Info | Edge Created |
|---|---|---|
| `@RestController`, `@Controller` | layer=controller | - |
| `@Service` | layer=service | - |
| `@Repository`, `@Mapper` | layer=repository | - |
| `@Autowired`, constructor injection | Dependency target | `DEPENDS_ON(injection)` |
| `@RequestMapping`, `@GetMapping`, etc. | HTTP path + method | `Method.http_path` property |
| `@Transactional` | Transactional method | Metadata tag (reference for Phase 2) |
| `@EventListener` | Event listener | Metadata tag (reference for Phase 2) |
| `@Bean` | Bean definition | `PROVIDES_BEAN` edge |

**Completion Criteria:**
```python
# Analyze BillController
meta = analyzer.analyze(controller_parse_result)
assert meta.bean_type == "controller"
assert "BillService" in meta.injected_dependencies
assert any(m.path == "/api/bills" for m in meta.request_mappings)
```

### 2-3. Neo4j Schema Design

> Detailed schema: see [neo4j-schema.md](./neo4j-schema.md)

**Key points:**
- **Pre-reserve** Phase 4 Ariadne bridge node types (`RDS`, `ECS`, `S3`, etc.)
- Distinguish static/runtime origin via the `source` property
- Indexes: `(node_id, project_id)` composite index

### 2-4. Graph Builder (Basic)

**Tasks:**
```python
# theseus/graph/builder.py

class GraphBuilder:
    """Parse results -> NetworkX -> Neo4j batch store"""

    def build(self, parse_results: list[ParseResult], spring_metadata: dict) -> nx.DiGraph:
        """Convert parse results to a NetworkX graph"""
        ...

    def store(self, graph: nx.DiGraph, project_id: str) -> StoreResult:
        """Batch store a NetworkX graph into Neo4j"""
        ...

@dataclass
class StoreResult:
    nodes_created: int
    edges_created: int
    errors: list[str]
    duration_seconds: float
```

**All nodes/edges the builder must create (Codex review revision):**
- **Nodes**: Class, Method (from ParseResult), Table (from MyBatisResult), ExternalAPI (from ConfigResult), Config (from ConfigResult)
- **Edges**: HAS_METHOD, CALLS, DEPENDS_ON (from ParseResult+SpringMetadata), QUERIES (from MyBatisResult), CALLS_API (from ConfigResult), USES_CONFIG (from SpringMetadata)

```python
def build(self, parse_results, spring_metadata, mybatis_results, config_results) -> nx.DiGraph:
    """Combine all parser results to create the graph"""
    graph = nx.DiGraph()
    # 1. Class + Method nodes (parse_results)
    # 2. DEPENDS_ON edges based on Spring annotations (spring_metadata)
    # 3. Table nodes + QUERIES edges (mybatis_results)
    # 4. ExternalAPI + Config nodes (config_results)
    return graph
```

**Batch store strategy (based on Potpie):**
- `UNWIND` batches of 1000 nodes
- Create all nodes first -> then create edges (2-pass)
- `CREATE INDEX IF NOT EXISTS` for indexes

**Completion Criteria:**
- Toy project -> stored successfully in Neo4j
- Verify graph visualization in Neo4j Browser

### 2-5. Integration Tests

```python
# tests/test_graph/test_integration.py

class TestGraphIntegration:
    """Toy project -> parse -> store in Neo4j -> Cypher query E2E"""

    def test_full_pipeline(self, neo4j_session, sample_app_path):
        # 1. Parse
        results = parser.parse_directory(sample_app_path)
        # 2. Spring analysis
        metadata = {r.file_path: analyzer.analyze(r) for r in results}
        # 3. Build graph + store
        graph = builder.build(results, metadata)
        store_result = builder.store(graph, project_id="test")
        # 4. Cypher verification
        assert store_result.nodes_created >= 6   # 6+ classes
        assert store_result.edges_created >= 5   # 5+ dependencies

    def test_query_dependencies(self, neo4j_session):
        """Query BillService dependencies"""
        result = neo4j_session.run("""
            MATCH (s:Class {name: 'BillService'})-[:DEPENDS_ON]->(dep)
            RETURN dep.name
        """)
        deps = [r["dep.name"] for r in result]
        assert "BillMapper" in deps
        assert "NotificationService" in deps
```

---

## Week 3: MyBatis/Config Parser + Incremental Parsing

### 3-1. MyBatis XML Parser

**Tasks:**
```python
# theseus/parser/mybatis_parser.py

class MyBatisParser:
    """MyBatis mapper XML -> SQL -> table reference extraction"""

    def parse_mapper(self, xml_path: str) -> MyBatisResult:
        """Parse a mapper XML"""
        ...

    def extract_tables(self, sql: str) -> list[str]:
        """Extract table names from SQL (regex-based)"""
        ...

@dataclass
class MyBatisResult:
    namespace: str                      # Mapper interface FQN
    statements: list[MyBatisStatement]  # select/insert/update/delete

@dataclass
class MyBatisStatement:
    id: str            # Method name
    type: str          # select | insert | update | delete
    sql: str           # Static SQL text
    tables: list[str]  # Referenced table names
    is_dynamic: bool   # Whether dynamic SQL tags like <if>, <foreach> are present
```

**SQL table extraction strategy:**
```python
TABLE_PATTERN = re.compile(
    r'\b(?:FROM|JOIN|INTO|UPDATE)\s+(\w+)',
    re.IGNORECASE
)

def extract_tables(self, sql: str) -> list[str]:
    # 1. Remove dynamic SQL tags (test attributes of <if>, <foreach>, etc.)
    clean_sql = re.sub(r'<[^>]+>', ' ', sql)
    # 2. Extract table names via regex
    return list(set(TABLE_PATTERN.findall(clean_sql)))
```

**Scope:**

| Type | Handled | Notes |
|---|---|---|
| Static SQL (`SELECT ... FROM bill`) | Yes | Table extraction via regex |
| `<if test="...">` | Partial | Strip tags then extract SQL; conditions ignored |
| `<foreach>` | Partial | Strip tags then extract SQL |
| `<include refid="...">` | Yes | Resolve refid references |
| `${variable}` dynamic table names | No | To be supplemented with Phase 2 runtime data |

### 3-2. Config Parser

```python
# theseus/parser/config_parser.py

class ConfigParser:
    """application.yml -> external service/DB config extraction"""

    def parse(self, yml_path: str) -> ConfigResult:
        ...

@dataclass
class ConfigResult:
    db_urls: list[DBConfig]          # spring.datasource.url, etc.
    external_apis: list[APIConfig]   # External API URL settings
    server_port: int | None
    profiles: list[str]              # spring.profiles

@dataclass
class DBConfig:
    key: str       # spring.datasource.url
    url: str       # jdbc:mysql://localhost:3306/dongne
    db_name: str   # dongne (extracted from URL)

@dataclass
class APIConfig:
    key: str       # payment.thepay.api-url
    url: str       # https://api.thepay.co.kr
    name: str      # thepay (extracted from key)
```

### 3-3. Parse Error Recovery

```python
# theseus/parser/tree_sitter_parser.py (extended)

def parse_directory(self, dir_path: str) -> list[ParseResult]:
    results = []
    for java_file in Path(dir_path).rglob("*.java"):
        try:
            result = self.parse_file(str(java_file))
            results.append(result)
        except Exception as e:
            logger.warning(f"Parse failed: {java_file}: {e}")
            results.append(ParseResult(
                file_path=str(java_file),
                classes=[], methods=[], imports=[], calls=[],
                errors=[ParseError(file=str(java_file), message=str(e))]
            ))
            # Record error to DB
            self._log_error(project_id, str(java_file), e)
    return results
```

### 3-4. Incremental Parsing

```python
# theseus/graph/builder.py (extended)

class IncrementalBuilder:
    """Incremental builder that only re-parses changed files"""

    def detect_changes(self, project_id: str, repo_path: str) -> ChangeSet:
        """Detect changed files via Git diff + file hashes"""
        # Primary: git diff --name-only HEAD~1 (fast detection)
        # Fallback: per-file SHA-256 hash vs DB cache (when git is unavailable)
        ...

    def incremental_parse(self, project_id: str, changes: ChangeSet) -> StoreResult:
        """Re-parse only changed files -> update Neo4j"""
        # 1. Deleted files -> remove related nodes/edges from Neo4j
        # 2. Modified/added files -> re-parse -> update existing nodes
        # 3. Update file hash cache (PostgreSQL)
        ...

@dataclass
class ChangeSet:
    added: list[str]
    modified: list[str]
    deleted: list[str]
```

**PG <-> Neo4j Synchronization Strategy:**
```
1. PG: Record project.status = 'parsing'
2. PG: Update file_hashes (store new hashes)
3. Neo4j: Batch store nodes/edges
4. PG: Set project.status = 'ready', last_parsed_at = now()
5. On failure: Set PG status = 'error', record in parse_errors table
```

### 3-5. PG <-> Neo4j Synchronization

Implement the synchronization strategy above. The key principle is to treat **PG as the source of truth** and Neo4j as derived data.

---

## Week 4: dongne-v2 Extension + CLI + Validation

### 4-1. dongne-v2 Parsing

**Expected edge cases:**
- Large files (1000+ lines)
- Lombok annotations (`@Getter`, `@Builder` -- tree-sitter recognizes them only as annotations)
- Multi-module Gradle projects
- Deep package structures (5+ depth)

### 4-2. CLI Tool

```python
# theseus/cli.py (Click-based)

@click.group()
def cli():
    """Theseus -- Code Topology MCP Server"""
    pass

@cli.command()
@click.argument('repo_path')
@click.option('--project-name', '-n', default=None)
@click.option('--incremental/--full', default=True)
def parse(repo_path, project_name, incremental):
    """Parse a codebase and generate a Neo4j graph"""
    ...

@cli.command()
@click.argument('project_name')
@click.option('--query', '-q', required=True)
def query(project_name, query):
    """Execute a Cypher query"""
    ...

@cli.command()
@click.argument('project_name')
def stats(project_name):
    """Print project graph statistics"""
    # Node/edge counts, class/method counts, table counts, etc.
    ...
```

### 4-3. Base Cypher Queries

```python
# theseus/graph/queries.py

QUERIES = {
    "class_dependencies": """
        MATCH (c:Class {name: $class_name, project_id: $pid})-[:DEPENDS_ON]->(dep)
        RETURN dep.name, dep.type, dep.file_path
    """,

    "table_users": """
        MATCH (m:Method)-[:QUERIES]->(t:Table {name: $table_name, project_id: $pid})
        MATCH (c:Class)-[:HAS_METHOD]->(m)
        RETURN c.name, m.name, c.type
    """,

    "call_chain": """
        MATCH path = (start:Method {name: $method_name})-[:CALLS*1..3]->(end)
        WHERE start.project_id = $pid
        RETURN path
    """,

    "dead_code_candidates": """
        MATCH (m:Method {project_id: $pid})
        WHERE NOT ()-[:CALLS]->(m)
        AND NOT m.name IN ['main', 'init', 'run']
        RETURN m.name, m.class_name, m.file_path
        ORDER BY m.class_name
    """,
}
```

### 4-4. Parsing Accuracy Validation

**Method:**
1. Randomly sample 10 classes from dongne-v2
2. **Manual analysis** of each class (human creates the dependency list)
3. Compare with Theseus parser output
4. Calculate Precision/Recall

| Metric | Calculation | Target |
|---|---|---|
| Precision | (correctly extracted dependencies) / (total extracted dependencies) | 90%+ |
| Recall | (correctly extracted dependencies) / (total actual dependencies) | 80%+ |
| Parse Success Rate | (successfully parsed files) / (total Java files) | 95%+ |

---

## Checklist

### Week 1
- [ ] `pyproject.toml` + uv environment setup
- [ ] ruff + pytest configuration
- [ ] Docker Compose (Neo4j + PostgreSQL) running
- [ ] PostgreSQL schema (projects, file_hashes, parse_errors) migration
- [ ] Toy Spring project created (6+ Java files, MyBatis XML, application.yml)
- [ ] tree-sitter Java parser basic implementation (class/method/import/calls)
- [ ] 5+ unit tests passing

### Week 2
- [ ] tree-sitter edge case handling (generics, lambdas, nested classes, enums)
- [ ] Spring annotation analyzer (`@Controller`, `@Service`, `@Repository`, `@Autowired`, etc.)
- [ ] Neo4j schema design + index creation (including Ariadne bridge node reservation)
- [ ] Graph builder (NetworkX -> Neo4j batch store)
- [ ] Integration test: toy project -> store in Neo4j -> Cypher query

### Week 3
- [ ] MyBatis XML parser (static SQL + regex table extraction)
- [ ] Config parser (application.yml -> DB/API settings)
- [ ] Parse error recovery (per-file try/except, partial graph guarantee)
- [ ] Incremental parsing (Git diff + SHA-256 hash cache)
- [ ] PG <-> Neo4j synchronization strategy implementation

### Week 4
- [ ] dongne-v2 parsing successful (100+ nodes, 200+ edges)
- [ ] CLI tool (`theseus parse`, `theseus query`, `theseus stats`)
- [ ] 4+ base Cypher queries working
- [ ] Parsing accuracy: Precision 90%+, Recall 80%+, Parse Success Rate 95%+
- [ ] Remaining graph generated normally even when some files fail to parse

### Phase 1 Gate (Criteria for Entering Next Phase)
- [ ] **Required**: Toy project -> Neo4j graph generation E2E working
- [ ] **Required**: dongne-v2 parsing -> meaningful graph
- [ ] **Required**: `theseus parse` CLI working
- [ ] **Recommended**: Incremental parsing working
- [ ] **Recommended**: Parsing accuracy targets met
