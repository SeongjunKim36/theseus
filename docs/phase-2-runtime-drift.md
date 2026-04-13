# Phase 2: Runtime Drift Layer — Detailed Specification

> Duration: Week 5–8 (4 weeks) · Prerequisite: Phase 1 complete, DEVELOPMENT_PLAN.md §3 Phase 2

---

## Goal

Collect runtime call data using the OpenTelemetry Java Agent,
assign call frequency weights to the static graph,
and **detect drift** between static analysis and runtime behavior.

---

## Week 5: OTel Collector + Trace Collection

### 5-1. OTel Collector Setup

**Docker Compose extension:**
```yaml
  otel-collector:
    image: otel/opentelemetry-collector-contrib:latest
    command: ["--config=/etc/otel-collector-config.yaml"]
    volumes:
      - ./otel/collector-config.yaml:/etc/otel-collector-config.yaml
    ports:
      - "4317:4317"   # OTLP gRPC
      - "4318:4318"   # OTLP HTTP
    depends_on:
      - postgres
```

**Collector configuration:**
```yaml
# otel/collector-config.yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318

processors:
  batch:
    timeout: 5s
    send_batch_size: 1024

exporters:
  otlphttp:
    endpoint: http://theseus-api:8000/api/traces  # Theseus receiver endpoint
  logging:
    loglevel: debug  # For development

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [batch]
      exporters: [otlphttp, logging]
```

### 5-2. Toy App Instrumentation

**Java Agent configuration:**
```bash
java -javaagent:opentelemetry-javaagent.jar \
     -Dotel.service.name=sample-spring-app \
     -Dotel.exporter.otlp.endpoint=http://localhost:4317 \
     -Dotel.traces.exporter=otlp \
     -Dotel.instrumentation.spring-web.enabled=true \
     -Dotel.instrumentation.jdbc.enabled=true \
     -Dotel.instrumentation.mybatis.enabled=true \
     -jar sample-spring-app.jar
```

**Span types to collect:**

| Span Type | Auto-instrumented | Extracted Info |
|---|---|---|
| HTTP request | Yes (spring-web) | `http.method`, `http.route`, `http.status_code` |
| DB query | Yes (jdbc) | `db.statement`, `db.name`, `db.system` |
| External HTTP call | Yes (http-client) | `http.url`, `http.method` |
| Method call | Partial (@WithSpan) | `code.namespace`, `code.function` |
| MyBatis | Yes (mybatis) | mapper + method |

### 5-3. Trace Parser

```python
# theseus/runtime/otel_collector.py

class TraceReceiver:
    """OTLP trace receiver + parser"""

    async def receive_traces(self, trace_data: dict) -> list[ParsedSpan]:
        """OTLP JSON -> structured span list"""
        ...

@dataclass
class ParsedSpan:
    trace_id: str
    span_id: str
    parent_span_id: str | None
    service_name: str
    operation_name: str
    span_kind: str            # SERVER, CLIENT, INTERNAL

    # For code mapping
    code_namespace: str | None   # com.example.service.BillService
    code_function: str | None    # createBill

    # For HTTP mapping
    http_method: str | None
    http_route: str | None       # /api/bills

    # For DB mapping
    db_statement: str | None     # SELECT * FROM bill WHERE ...
    db_name: str | None

    # For external API mapping
    http_url: str | None         # https://api.thepay.co.kr/pay

    timestamp: datetime
    duration_ms: float
```

### 5-4. Span -> Code Node Mapping

```python
# theseus/runtime/trace_mapper.py

class TraceMapper:
    """OTel span -> Neo4j code node mapping"""

    def map_span(self, span: ParsedSpan, project_id: str) -> MappingResult | None:
        """Map a span to a Neo4j node"""

        # Strategy 1: Direct mapping via code.namespace + code.function
        if span.code_namespace and span.code_function:
            return self._match_by_code_attr(span, project_id)

        # Strategy 2: HTTP route -> Controller method mapping
        if span.http_route and span.span_kind == "SERVER":
            return self._match_by_http_route(span, project_id)

        # Strategy 3: DB statement -> MyBatis method mapping
        if span.db_statement:
            return self._match_by_sql(span, project_id)

        # Strategy 4: External HTTP URL -> ExternalAPI node mapping
        if span.http_url and span.span_kind == "CLIENT":
            return self._match_by_external_url(span, project_id)

        return None

    def _match_by_code_attr(self, span, project_id) -> MappingResult:
        """Search Neo4j for a node matching code.namespace (FQN) + code.function"""
        # Codex revision: code_namespace is an FQN, so match against Class.fqn
        result = self.neo4j.run("""
            MATCH (c:Class {fqn: $namespace, project_id: $pid})-[:HAS_METHOD]->(m:Method {name: $function})
            RETURN m.node_id
        """, namespace=span.code_namespace, function=span.code_function, pid=project_id)
        ...

    def _match_by_http_route(self, span, project_id) -> MappingResult:
        """Match against @RequestMapping paths"""
        result = self.neo4j.run("""
            MATCH (m:Method {project_id: $pid})
            WHERE m.http_path = $route
            RETURN m.node_id
        """, route=span.http_route, pid=project_id)
        ...

@dataclass
class MappingResult:
    span: ParsedSpan
    node_id: str
    confidence: float       # Mapping confidence (0.0 ~ 1.0)
    strategy: str           # code_attr | http_route | sql | external_url
```

**Mapping confidence levels:**

| Strategy | Confidence | Notes |
|---|---|---|
| code.namespace + function | 0.95 | Direct mapping |
| HTTP route -> Controller | 0.85 | Path pattern matching |
| SQL -> MyBatis | 0.70 | SQL similarity-based |
| External URL | 0.90 | URL prefix matching |

---

## Week 6: Runtime Weights + Drift Detection

### 6-1. Neo4j Schema Extension

```cypher
// Add runtime properties to existing CALLS edges
// source: "static" | "runtime" | "both"
// runtime_count: cumulative call count
// last_called_at: timestamp of last call
// first_seen_at: timestamp of first discovery

MATCH (a:Method)-[r:CALLS]->(b:Method)
SET r.runtime_count = COALESCE(r.runtime_count, 0),
    r.source = COALESCE(r.source, "static"),
    r.last_called_at = null,
    r.first_seen_at = null
```

### 6-1b. Span Chain Assembly (Codex Revision)

> **Problem**: Individual spans only identify the callee. The caller->callee relationship must be assembled via `parent_span_id`.

```python
# theseus/runtime/trace_mapper.py (extended)

class TraceChainAssembler:
    """Convert span chains into caller->callee edge lists based on parent_span_id"""

    def assemble(self, spans: list[ParsedSpan]) -> list[tuple[MappingResult, MappingResult]]:
        """Build span tree per trace_id -> extract caller/callee pairs"""
        span_map = {s.span_id: s for s in spans}
        edges = []
        for span in spans:
            if span.parent_span_id and span.parent_span_id in span_map:
                parent = span_map[span.parent_span_id]
                caller_mapping = self.mapper.map_span(parent, self.project_id)
                callee_mapping = self.mapper.map_span(span, self.project_id)
                if caller_mapping and callee_mapping:
                    edges.append((caller_mapping, callee_mapping))
        return edges
```

### 6-2. Runtime Weight Assignment

```python
# theseus/runtime/otel_collector.py (extended)

class RuntimeWeightUpdater:
    """Mapped spans -> Neo4j edge weight updates"""

    def update_weights(self, mappings: list[MappingResult]):
        """Batch update edge weights"""
        for mapping in mappings:
            self.neo4j.run("""
                MATCH (caller)-[r:CALLS]->(callee {node_id: $node_id})
                SET r.runtime_count = COALESCE(r.runtime_count, 0) + 1,
                    r.last_called_at = datetime($timestamp),
                    r.source = CASE WHEN r.source = 'static' THEN 'both' ELSE r.source END
            """, node_id=mapping.node_id, timestamp=mapping.span.timestamp)

    def create_runtime_only_edges(self, trace_chain: list[MappingResult]):
        """Create runtime call edges not present in the static graph"""
        for i in range(len(trace_chain) - 1):
            caller = trace_chain[i]
            callee = trace_chain[i + 1]
            self.neo4j.run("""
                MATCH (a {node_id: $caller_id}), (b {node_id: $callee_id})
                MERGE (a)-[r:CALLS]->(b)
                ON CREATE SET r.source = 'runtime', r.runtime_count = 1,
                              r.first_seen_at = datetime($ts)
                ON MATCH SET r.runtime_count = r.runtime_count + 1,
                             r.last_called_at = datetime($ts)
            """, caller_id=caller.node_id, callee_id=callee.node_id, ts=caller.span.timestamp)
```

### 6-3. Drift Detection Engine

```python
# theseus/runtime/drift_detector.py

class DriftDetector:
    """Detect discrepancies between static analysis and runtime data"""

    def detect(self, project_id: str, min_days: int = 14) -> DriftReport:
        static_only = self._find_static_only(project_id, min_days)
        runtime_only = self._find_runtime_only(project_id)
        frequency_anomalies = self._find_frequency_anomalies(project_id)
        return DriftReport(static_only, runtime_only, frequency_anomalies)

    def _find_static_only(self, project_id, min_days) -> list[DriftItem]:
        """Edges that exist in static analysis but have not been called at runtime for N days"""
        # Codex revision: project_id is on nodes, not edges
        return self.neo4j.run("""
            MATCH (a {project_id: $pid})-[r:CALLS {source: 'static'}]->(b)
            WHERE r.last_called_at IS NULL
               OR r.last_called_at < datetime() - duration({days: $days})
            RETURN a.name AS caller, b.name AS callee,
                   r.last_called_at, b.file_path
            ORDER BY r.last_called_at ASC NULLS FIRST
        """, pid=project_id, days=min_days)

    def _find_runtime_only(self, project_id) -> list[DriftItem]:
        """Edges called at runtime but absent from static analysis (reflection/AOP/proxy)"""
        return self.neo4j.run("""
            MATCH (a)-[r:CALLS {source: 'runtime', project_id: $pid}]->(b)
            RETURN a.name AS caller, b.name AS callee,
                   r.runtime_count, r.first_seen_at
            ORDER BY r.runtime_count DESC
        """, pid=project_id)

@dataclass
class DriftReport:
    static_only: list[DriftItem]      # Dead code candidates
    runtime_only: list[DriftItem]     # Hidden paths (AOP/reflection)
    frequency_anomalies: list[DriftItem]  # Abnormal call frequencies
    generated_at: datetime

@dataclass
class DriftItem:
    caller: str
    callee: str
    drift_type: str    # STATIC_ONLY | RUNTIME_ONLY | FREQUENCY_ANOMALY
    details: dict      # last_called_at, runtime_count, etc.
    file_path: str | None
    severity: str      # high | medium | low
```

### 6-4. Golden Dataset Validation

**Insert intentional drift into the toy app:**

| Scenario | Toy App Setup | Expected Detection |
|---|---|---|
| Dead code | `BillService.legacyRefund()` -- exists in code but never called | `STATIC_ONLY` |
| AOP hidden path | `@Transactional` -> proxy actually calls `TransactionInterceptor` | `RUNTIME_ONLY` |
| High-frequency path | `BillService.createBill()` -- called 100 times in tests | Weight 100 |
| Low-frequency path | `BillService.getBill()` -- called 1 time in tests | Weight 1 |

```python
# tests/test_runtime/test_drift_golden.py

class TestDriftGolden:
    """Drift detection accuracy validation based on golden dataset"""

    def test_detects_dead_code(self, drift_detector, project_with_runtime):
        report = drift_detector.detect(project_with_runtime, min_days=0)
        dead_methods = [d.callee for d in report.static_only]
        assert "legacyRefund" in dead_methods

    def test_detects_aop_hidden_path(self, drift_detector, project_with_runtime):
        report = drift_detector.detect(project_with_runtime)
        runtime_callers = [d.caller for d in report.runtime_only]
        # Detect TransactionInterceptor or proxy-related paths
        assert any("proxy" in c.lower() or "interceptor" in c.lower() for c in runtime_callers)

    def test_runtime_weights(self, neo4j_session, project_with_runtime):
        result = neo4j_session.run("""
            MATCH ()-[r:CALLS {source: 'both'}]->({name: 'createBill'})
            RETURN r.runtime_count
        """)
        assert result.single()["r.runtime_count"] >= 50
```

---

## Weeks 7–8: dongne-v2 Staging Deployment

### 7-1. Staging Environment OTel Setup

**Prerequisites (request during Week 1):**
- [ ] Access to dongne-v2 staging environment
- [ ] Secure Java Agent JAR deployment path
- [ ] Verify Collector endpoint network connectivity

**Minimizing performance impact:**
```bash
# Limit CPU overhead to 2-3% via sampling
-Dotel.traces.sampler=parentbased_traceidratio
-Dotel.traces.sampler.arg=0.1  # 10% sampling
```

### 7-2 ~ 7-4. Data Accumulation + Drift Analysis + Validation

- Collect traces for at least **1-2 weeks** (work on Phase 3 in parallel during this period)
- Generate drift report after collection
- Cross-check with dongne-v2 team: "Is this method really unused?"

---

## Checklist

### Week 5
- [ ] Add OTel Collector to Docker Compose
- [ ] Collector config (OTLP receiver -> Theseus exporter)
- [ ] Attach OTel Java Agent to toy app -> verify span reception
- [ ] TraceReceiver: OTLP JSON -> ParsedSpan conversion
- [ ] TraceMapper: 4 strategies (code_attr, http_route, sql, external_url)
- [ ] Mapping unit tests (toy app-based)

### Week 6
- [ ] Neo4j schema extension (runtime_count, last_called_at, source)
- [ ] RuntimeWeightUpdater: span -> edge weight update
- [ ] Runtime-only edge creation (MERGE)
- [ ] DriftDetector: STATIC_ONLY, RUNTIME_ONLY detection
- [ ] Golden dataset validation: dead code detection, AOP hidden path detection
- [ ] `theseus drift-report` CLI working

### Weeks 7–8
- [ ] dongne-v2 staging OTel setup complete
- [ ] At least 1 week of trace data accumulated
- [ ] dongne-v2 drift report generated
- [ ] 3+ dead code candidates found
- [ ] 1+ RUNTIME_ONLY edge found
- [ ] Cross-check with dongne-v2 team complete

### Phase 2 Gate
- [ ] **Required**: Toy app OTel -> Neo4j weight E2E working
- [ ] **Required**: Drift detection golden tests passing
- [ ] **Required**: `theseus drift-report` CLI working
- [ ] **Recommended**: Actual drift discovered in dongne-v2
- [ ] **Recommended**: 80%+ of mappings with confidence >= 0.7
