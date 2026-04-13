# Theseus — 개발 계획서

> 작성: 2026-04-13 · 수정: 2026-04-13 (Codex 리뷰 반영) · 기반 문서: [기획서](./project-b-code-mapper.md)

---

## 0. 오픈소스 분석 결과: Fork vs Reference

### Potpie AI 코드베이스 해부

Potpie AI(GitHub: `potpie-ai/potpie`, Apache-2.0, 5300+ stars)의 실제 코드를 분석한 결과:

| 모듈 | 내용 | Theseus 활용도 |
|---|---|---|
| `app/modules/parsing/graph_construction/` | **RepoMap**(Aider 기반) + tree-sitter → Neo4j 배치 저장 | 핵심 참고 |
| `app/modules/parsing/knowledge_graph/` | SentenceTransformer(`all-MiniLM-L6-v2`) 임베딩 + LLM 독스트링 | 참고 |
| `app/modules/intelligence/agents/` | BlastRadiusAgent, DebugAgent 등 pydantic-ai 기반 | 패턴 참고 |
| `app/modules/intelligence/tools/` | KG 질의, 코드 분석, change detection 도구 | 패턴 참고 |
| `app/modules/auth/`, `billing/`, `analytics/` | Firebase, Stripe, PostHog 등 SaaS 인프라 | **불필요** |
| `compose.yaml` | PostgreSQL + Neo4j + Redis | 인프라 참고 |

### 결정: **Reference (참고), Fork 아님**

**이유:**

1. **핵심 파싱 로직이 얇다** — 실제 파싱은 `RepoMap`(Aider에서 가져온 ~500줄) + tree-sitter + Neo4j 배치 저장이 전부. 이걸 위해 전체 SaaS를 포크하는 건 과잉
2. **불필요한 의존성 과다** — Firebase, Stripe, GCP Secret Manager, Celery, Redis, LangChain, LangGraph 등. Theseus에 필요 없는 것이 80%
3. **포크 유지보수 부담** — 134개 오픈 이슈, 빠른 업데이트 속도. 포크하면 upstream 추적이 짐이 됨
4. **MCP 서버 없음** — Potpie에는 MCP가 없어서 어차피 직접 구현해야 함
5. **런타임 레이어 없음** — Theseus 핵심 차별점을 어차피 직접 만들어야 함

### 가져올 것 (Reference)

```
Potpie에서 참고                          Theseus에서 직접 구현
──────────────────────────────          ──────────────────────────
tree-sitter Java 태그 쿼리              → Spring/MyBatis 특화 파서
  (.scm 파일)
RepoMap 패턴 (그래프 구축 흐름)          → 경량화된 그래프 빌더
Neo4j 스키마/인덱싱 전략                 → 런타임 가중치 확장 스키마
BlastRadiusAgent 프롬프트 패턴           → MCP tool로 재구현
SentenceTransformer 임베딩 패턴          → 로컬 임베딩 + Claude 하이브리드
compose.yaml 인프라 구성                 → 최소 구성 (Neo4j + PostgreSQL)
```

---

## 1. 기술 스택 확정

| 레이어 | 기술 | 선택 근거 |
|---|---|---|
| 언어 | **Python 3.12+** | Potpie 참고 코드와 동일, tree-sitter 생태계, MCP SDK 지원 |
| 웹 프레임워크 | **FastAPI** | 비동기, 타입 힌트, MCP 서버와 공존 가능 |
| 정적 파싱 | **tree-sitter** + `tree-sitter-java` | Potpie 검증 완료, 다국어 확장 용이 |
| Spring 특화 | **직접 구현** (tree-sitter 위에) | `@Service`, `@Controller`, MyBatis XML, `application.yml` 파싱 |
| 그래프 DB | **Neo4j 5.x** + APOC | Potpie/Ariadne 공통, 그래프 트래버설 성능 |
| ��타데이터 DB | **PostgreSQL 16** | 프로젝트 설정, 파싱 상태, 캐시 |
| 임베딩 | **SentenceTransformer** (`all-MiniLM-L6-v2`) | 로컬 실행, 무료, Potpie 검증 완료 |
| LLM | **Claude API** (`claude-sonnet-4-6`) | 의미 분석, 서브그래프 요약 |
| MCP 서버 | **`mcp` Python SDK** (`fastmcp`) | 공식 SDK, Claude Code 네이티브 연결 |
| 런타임 수집 | **OpenTelemetry Java Agent** + **OTLP Collector** | 표준 프로토콜, 벤더 무관 |
| 컨테이너 | **Docker Compose** | 로컬 개발, Neo4j + PostgreSQL + OTel Collector |
| 프론트 (Phase 5) | **React + Cytoscape.js** | 최소 웹 뷰어, MCP가 주 인터페이스 |

---

## 2. 프로젝트 구조 (목표)

```
theseus/
├��─ docker-compose.yml          # Neo4j + PostgreSQL + OTel Collector
├── pyproject.toml
├── README.md
│
├── theseus/
│   ├── __init__.py
│   ├── config.py               # 환경 변수, 설정
│   │
│   ├── parser/                 # Phase 1: 정적 분석
│   │   ├── __init__.py
│   │   ├── tree_sitter_parser.py    # tree-sitter 기반 AST 파싱
│   │   ├── spring_analyzer.py       # Spring 빈/어노테이션 분석
│   │   ├── mybatis_parser.py        # MyBatis XML → SQL → 테이블
│   │   ├── config_parser.py         # application.yml 파싱
│   │   └── queries/                 # tree-sitter 쿼리 (.scm)
│   │       └── java-tags.scm
│   │
│   ├── graph/                  # Phase 1: 그래프 구축/저장
│   │   ├── __init__.py
│   │   ├── builder.py          # 파싱 결과 → NetworkX → Neo4j
│   │   ├── schema.py           # Neo4j 노드/엣지 타입 정의
│   │   ├── neo4j_client.py     # Neo4j 드라이버 래퍼
│   │   └── queries.py          # Cypher 쿼리 모음
│   │
│   ├── runtime/                # Phase 2: 런타임 수집
│   │   ├── __init__.py
│   │   ├── otel_collector.py   # OTLP → 파싱 → 그래프 가중치
│   │   ├── trace_mapper.py     # span → 코드 노드 매핑
│   │   └── drift_detector.py   # 정적 vs 런타임 드리프트
│   │
│   ├── mcp/                    # Phase 3: MCP 서버
│   │   ├── __init__.py
│   │   ├── server.py           # FastMCP 서버 정의
│   │   └── tools.py            # MCP tool 구현 (5개)
│   │
│   ├── intelligence/           # Phase 3: AI 의미층
│   │   ├── __init__.py
│   │   ├── embeddings.py       # SentenceTransformer 임베딩
│   │   ├── query_engine.py     # 자연어 → Cypher 변환
│   │   └── summarizer.py       # 서브그래프 → LLM 요약
│   │
│   ├── bridge/                 # Phase 4: Ariadne 브릿지
│   │   ├── __init__.py
│   │   └── ariadne_connector.py
│   │
│   └── api/                    # REST API (보조)
│       ├── __init__.py
│       └── router.py
│
├── web/                        # Phase 5: 최소 웹 뷰어
│   └── (React + Cytoscape.js)
│
├── otel/                       # OTel Collector 설정
│   └── collector-config.yaml
│
├── tests/
│   ├── fixtures/               # 장난감 Spring 프로젝트
│   │   └── sample-spring-app/
│   ├── test_parser/
│   ├── test_graph/
│   ├── test_runtime/
│   └── test_mcp/
│
└── docs/
    ├── project-b-code-mapper.md
    └── DEVELOPMENT_PLAN.md
```

---

## 3. Phase별 상세 개발 계획

---

### Phase 1: 정적 그래프 MVP (4주) ← Codex 리뷰 반영, 3주→4주

**목표**: 장난감 Spring 프로젝트 → Neo4j 코드 그래프 생성 → dongne-v2로 확장

#### Week 1: 인프라 + tree-sitter 기본 파서 + 테스트 기반

| 작업 | 상세 | 산출물 |
|---|---|---|
| 1-1. 프로젝트 초기화 | Python 프로젝트(pyproject.toml, uv), linter(ruff), 테스트(pytest) | 빌드 가능한 빈 프로젝트 |
| 1-2. Docker Compose | Neo4j 5.x + APOC, PostgreSQL 16 | `docker-compose.yml` |
| 1-3. PostgreSQL 메타데이터 스키마 | 프로젝트 설정, 파싱 상태, 파일 해시 캐시 테이블 설계 | `alembic/` 마이그레이션 |
| 1-4. 장난감 Spring 프로젝트 | Controller → Service → Repository → MyBatis XML, 3~5개 클래스 | `tests/fixtures/sample-spring-app/` |
| 1-5. tree-sitter Java 파서 (기본) | Potpie의 `java-tags.scm` 참고, 클래스/메서드/import 추출. 제네릭/람다/중첩 클래스 엣지케이스는 W2로 분리 | `theseus/parser/tree_sitter_parser.py` |
| 1-6. 픽스처 ��반 단위 테스트 | 파서 출력 → 기대 노드/엣지 JSON 비교. 이후 모든 파서 변경의 리그레션 방지 기반 | `tests/test_parser/` |

#### Week 2: tree-sitter 안정화 + Spring 어노테��션

| 작업 | 상세 | 산출물 |
|---|---|---|
| 2-1. tree-sitter 엣지케이스 처리 | Java 제네릭(`List<Map<K,V>>`), 람다, 중첩 클래스, enum 등 | `tree_sitter_parser.py` 안정화 |
| 2-2. Spring 어노테이션 분석기 | `@Controller`, `@Service`, `@Repository`, `@Component`, `@Autowired`, `@Bean` 인식 | `spring_analyzer.py` |
| 2-3. Neo4j 스키마 설계 | 노드 타입 + 엣지 타입 + 인덱스. **Phase 4 Ariadne 브릿지 노드 타입도 미리 예약** (스키마 마이그레이션 방지) | `schema.py` |
| 2-4. 그래프 빌더 (기본) | tree-sitter 결과 → NetworkX → Neo4j 배치 저장 (Potpie 패턴 참고) | `builder.py` |
| 2-5. 통합 테스트 | 장난감 프로젝트 → Neo4j 저장 → Cypher 질의 검증 | `tests/test_graph/` |

#### Week 3: MyBatis/Config 파서 + Neo4j 완성

| 작업 | 상세 | 산출물 |
|---|---|---|
| 3-1. MyBatis XML 파서 | mapper XML → **정적 SQL만** 추출 → 테이블 참조 엣지. 동적 SQL(`<if>`, `<foreach>`) 내 Java 표현식은 별도 처리(정규식 기반 테이블명 추출) | `mybatis_parser.py` |
| 3-2. config 파서 | `application.yml` → 외부 서비스 URL/DB 설정 추출 | `config_parser.py` |
| 3-3. 파싱 오류 복��� | 개별 파일 파싱 실패 시 스킵 + 로그, 부분 그래프 생성 보장 | `parser/` 전반 |
| 3-4. 증분 파싱 | **Git diff 기반**: `git diff --name-only HEAD~1` → 변경 파일만 재파싱. 파일 해시(SHA-256)를 PostgreSQL에 캐시하여 중복 파싱 방지 | `graph/builder.py` |
| 3-5. PG ↔ Neo4j 동기화 | 파싱 상태/파일 해시는 PG, 그래프 데이터는 Neo4j. 파싱 트랜잭션 시 양쪽 정합성 보장(PG 먼저 기록 → Neo4j 저장 → PG 상태 업데이트) | 동기화 전략 |

#### Week 4: dongne-v2 확장 + CLI + 검증

| 작업 | 상세 | 산출물 |
|---|---|---|
| 4-1. dongne-v2 파싱 | 실제 코드베이스 투입, 엣지 케이스 수정 | 파서 안정화 |
| 4-2. CLI 도구 | `theseus parse <repo_path>` → Neo4j 그래프 생성 | CLI 엔트리포인트 |
| 4-3. 기본 Cypher 질의 | "BillService의 모든 의존성", "bill 테이블 사용하는 코드" 등 | `graph/queries.py` |
| 4-4. 파싱 정확도 검증 | dongne-v2에서 수동으로 10개 클래스 샘플링 → 파서 출력과 실제 의존성 비교 | 정확도 리포트 |

**Phase 1 종료 조건:**
- [ ] 장난감 Spring 프로젝트 → Neo4j 그래프 생성 성공
- [ ] dongne-v2 파싱 → 의미 있는 그래프 (노드 100개+, 엣지 200개+)
- [ ] `theseus parse` CLI 동작
- [ ] Cypher 질의로 "결제 관련 클래스" 추출 가능
- [ ] 파싱 실패 파일이 있어도 나머지 그래프 정상 생성 (오류 ���구)
- [ ] 10개 샘플 클래스 대상 파싱 정확도 80%+

**기술 리스크 & 대응:**

| 리스크 | 확률 | 대응 |
|---|---|---|
| tree-sitter가 Java 제네릭/중첩 클래스 파싱 실패 | 중간 | W2에 전용 시간 배정. Potpie의 `java-tags.scm` 이미 검증됨. 최악 시 JavaParser 병행 |
| MyBatis 동적 SQL(`<if>`, `<foreach>`) 파싱 어려움 | 높음 | 1차는 정적 SQL + 정규식 테이블명 추출만. 동적 SQL 정확도는 Phase 2 런타임으로 보완 |
| dongne-v2 규모에서 Neo4j 저장 느림 | 낮음 | 배치 저장(1000개 단위), 인덱싱 전략 적용 |
| 개별 파일 파싱 실패로 전체 중단 | 중간 | 파일별 try/except, 실패 파일 로그 후 계속 진행 |

---

### Phase 2: 런타임 드리프트 레이어 (3~4주)

**목표**: OpenTelemetry로 런타임 호출 수집 → 정적 그래프에 가중치 부여 → 드리프트 감지

#### Week 5: OTel Collector + 트레이스 수집

> **사전 조건**: dongne-v2 staging 환경 접근 허가를 Phase 1 시작 시점(Week 1)에 요청해야 함. Phase 2에서 실제 필요하므로 최소 4주 전에 인프라팀에 사전 협의.

| 작업 | 상세 | 산출물 |
|---|---|---|
| 5-1. OTel Collector 설정 | Docker Compose에 OTLP Collector 추가, OTLP gRPC receiver → 파일/DB exporter | `otel/collector-config.yaml` |
| 5-2. 장난감 앱 계측 | OTel Java Agent로 장난감 Spring 앱 실행 → 트레이스 확인 | 계측된 장난감 앱 |
| 5-3. 트레이스 파서 | OTLP span 수신 → HTTP 경로, DB 쿼리, 외부 API 호출 추출 | `runtime/otel_collector.py` |
| 5-4. span → 코드 노드 매핑 | span의 `code.namespace`, `code.function` → Neo4j 노드 ID 매핑. **프록시/AOP span은 `code.namespace` 누락 가능** → HTTP 경로 + SQL 텍스트 + span 계층으로 복합 매핑 | `runtime/trace_mapper.py` |

#### Week 6: 런타임 가중치 + 드리프트 감지

| 작업 | 상세 | 산출물 |
|---|---|---|
| 6-1. Neo4j 스키마 확장 | 엣지에 `runtime_count`, `last_called_at`, `source` 속성 추가 | `schema.py` 확장 |
| 6-2. 런타임 가중치 부여 | 트레이스 데이터 → 기존 정적 엣지에 호출 빈도 업데이트 | `otel_collector.py` 확장 |
| 6-3. 드리프트 감지 엔진 | 정적 엣지 vs 런타임 데이터 비교: `STATIC_ONLY`(데드 코드 후보), `RUNTIME_ONLY`(숨은 경로) | `runtime/drift_detector.py` |
| 6-4. 드리프트 리포트 + 검증 방법 | 분류별 리포트 (JSON/CLI). **골든 데이터셋**: 장난감 앱에서 의도적으로 데드 코드 + AOP 경로를 포함하여 드리프트 감지 정확도 검증 | CLI 확장 + 골든 테스트 |

#### Week 7~8: dongne-v2 Staging 적용

| 작업 | 상세 | 산출물 |
|---|---|---|
| 7-1. staging 환경 OTel 설정 | dongne-v2 staging에 Java Agent 부착, Collector 연결 | 운영 문서 |
| 7-2. 데이터 축적 | 최소 1~2주 트레이스 수집 (이 기간 동안 Phase 3 병행) | 런타임 데이터셋 |
| 7-3. 드리프트 분석 | 실제 데드 코드 후보, 숨은 AOP 경로 발견 | 드리프트 리포트 |
| 7-4. 검증 | 발견된 드리프트를 dongne-v2 팀과 크로스체크 | 정확도 리포트 |

**Phase 2 종료 조건:**
- [ ] 장난감 앱에서 OTel 트레이스 → Neo4j 엣지 가중치 반영 성공
- [ ] 골든 데이터셋 기반 드리프트 감지 정확도 검증 통과
- [ ] 드리프트 감지로 데드 코드 후보 3개+ 발견
- [ ] `RUNTIME_ONLY` 엣지에서 리플렉션/AOP 경로 1개+ 발견
- [ ] `theseus drift-report` CLI 동작

**기술 리스크 & 대응:**

| 리스크 | 확률 | 대응 |
|---|---|---|
| OTel span의 `code.namespace` 누락 (프록시/AOP) | 높음 | span 속성 + HTTP 경로 + SQL 텍스트로 복합 매핑 |
| staging 환경 접근 지연 | 중간 | 장난감 앱 검증을 먼저 완료, staging은 데이터 축적용으로만 |
| 트레이스 볼륨 과다 | 중간 | 샘플링 + 집계(span별 카운트만 저장, raw span 버림) |

---

### Phase 3: MCP 서버 + AI 의미층 (3주)

**목표**: Claude Code/Cursor에서 MCP로 연결하여 코드 토폴로지 질의 가능

> Phase 2의 staging 데이터 축적(Week 7~8)과 **병행 진행**

#### Week 7: MCP 서버 기본 + 임베딩

| 작업 | 상세 | 산출물 |
|---|---|---|
| 7-1. FastMCP 서버 셋업 | `mcp` Python SDK로 서버 골격, stdio/SSE 전송 | `mcp/server.py` |
| 7-2. `theseus/get_subgraph` | 진입점(클래스/메서드명) → N-hop 서브그래프 반환 (JSON) | `mcp/tools.py` |
| 7-3. `theseus/blast_radius` | 타겟 노드 → 다운스트림 영향 범위 계산 | `mcp/tools.py` |
| 7-4. `theseus/drift_report` | 드리프트 데이터 조회/반환 | `mcp/tools.py` |
| 7-5. 로컬 임베딩 | 코드 특화 임베딩 모델로 노드별 벡터 생성 → Neo4j 속성 저장. **모델 선택**: `all-MiniLM-L6-v2`(범용) vs `codebert-base`(코드 특화) 벤치마크 후 결정. 코드+주석 혼합이므로 둘 다 테스트 | `intelligence/embeddings.py` |

#### Week 8: 자연어 질의 + snapshot

| 작업 | 상세 | 산출물 |
|---|---|---|
| 8-1. 자연어 → 서브그래프 | 자연어 질의 → 임베딩 유사도 + 키워드 매칭 → 관련 노드 식별 → 서브그래프 확장 | `intelligence/query_engine.py` |
| 8-2. `theseus/snapshot` | 스코프(전체/모듈/서비스) → 압축된 토폴로지 요약 (AI 컨텍스트 주입용) | `mcp/tools.py` |
| 8-3. 서브그래프 요약 | Claude API로 서브그래프 → 자연어 설명 생성 | `intelligence/summarizer.py` |
| 8-4. `get_subgraph` 자연어 모드 | 진입점 대신 자연어 입력 시 query_engine 경유 | `mcp/tools.py` 확장 |

#### Week 9: Claude Code 통합 테스트

| 작업 | 상세 | 산출물 |
|---|---|---|
| 9-1. Claude Code 연결 | `claude_desktop_config.json` / `.claude/settings.json`에 Theseus MCP 등록 → 실제 질의 | 설정 가이드 |
| 9-2. MCP 안정성 | 타임아웃 처리, Neo4j 다운 시 graceful 에러, 대형 서브그래프 truncation (depth/노드 수 cap) | 안정화 |
| 9-3. E2E 시나리오 테스트 | "결제 관련 코드 보여줘" → 서브그래프 → AI 요약 전체 흐름 | 테스트 결과 |
| 9-4. 응답 최적화 | 토큰 사용량 측정, 스냅샷 압축, 불필요한 정보 제거. **MCP 응답 사이즈 vs 정보 밀도** 트레이드오프 조정 | 성능 리포트 |

**Phase 3 종료 조건:**
- [ ] Claude Code에서 `theseus/get_subgraph` 호출 → 정확한 서브그래프 반환
- [ ] "결제 관련 코드" 자연어 질의 → 관련 노드 5개+ 반환
- [ ] `theseus/blast_radius` → 다운스트림 영향 목록 반환
- [ ] 스냅샷이 AI 컨텍스트 윈도우 효율을 실제로 개선하는지 측정
- [ ] Neo4j 다운 등 장애 시 MCP 에러 응답 정상 반환 (크래시 아님)

**MCP Tool 상세 스펙:**

```python
# theseus/get_subgraph
@mcp.tool()
async def get_subgraph(
    query: str,           # 자연어 또는 노드명 (e.g., "결제", "BillService")
    depth: int = 2,       # 그래프 탐색 깊이
    include_runtime: bool = True,  # 런타임 가중치 포함 여부
) -> dict:
    """코드베이스에서 관련 서브그래프를 반환합니다."""
    ...

# theseus/blast_radius
@mcp.tool()
async def blast_radius(
    target: str,          # 파일 경로 또는 메서드명
    direction: str = "downstream",  # downstream | upstream | both
) -> dict:
    """변경 시 영향 받는 코드 범위를 분석합니다."""
    ...

# theseus/drift_report
@mcp.tool()
async def drift_report(
    scope: str = "all",   # all | module명
    min_days: int = 14,   # 최소 미사용 기간
) -> dict:
    """정적 분석 vs 런타임 데이터의 불일치를 보고합니다."""
    ...

# theseus/snapshot
@mcp.tool()
async def snapshot(
    scope: str = "all",   # all | module명 | service명
    format: str = "compact",  # compact | detailed
) -> dict:
    """AI 컨텍스트 주입용 압축 토폴로지 스냅샷을 반환합니다."""
    ...
```

**기술 리스크 & 대응:**

| 리스크 | 확률 | 대응 |
|---|---|---|
| 자연어 → 서브그래프 정확도 낮음 | 중간 | 임베딩 유사도 + 키워드 + 그래프 이웃 3단계 파이프라인 |
| MCP 응답이 너무 큼 (대형 서브그래프) | 높음 | depth 제한 + 노드 수 cap + compact format |
| MCP 프로토콜 스펙 변경 | 낮음 | fastmcp SDK가 추상화, REST API도 병행 노출 |

---

### Phase 4: Ariadne 브릿지 (2주)

**목표**: Theseus(코드 그래프) ↔ Ariadne(인프라 그래프) Neo4j 연결

> Ariadne 프로젝트가 Phase 1 이상 진행된 상태를 전제

#### Week 9: 브릿지 설계 + 구현

| 작업 | 상세 | 산출물 |
|---|---|---|
| 10-1. 브릿지 스키마 | 코드 노드 ↔ 인프라 노드 연결 규칙: DB 설정 → RDS, URL → ECS/ALB 등 | `bridge/ariadne_connector.py` |
| 10-2. 자동 매핑 | `application.yml`의 DB URL → Ariadne의 RDS 노드, API URL → ECS 태스크 | 매핑 엔진 |
| 10-3. `theseus/ariadne_bridge` MCP tool | 코드 노드 → 연결된 인프라 노드 반환 | `mcp/tools.py` 확장 |

#### Week 11: 검증 + 통합

| 작업 | 상세 | 산출물 |
|---|---|---|
| 11-1. E2E 테스트 | "BillMapper → RDS → VPC → 다른 ECS 태스크" 전경로 추적 | 테스트 결과 |
| 11-2. 역방향 질의 | "이 RDS에 의존하는 코드는?" → Ariadne → Theseus 역추적 | 양방향 질의 |
| 11-3. 통합 시각화 | Cytoscape.js에 코드+인프라 노드 동시 표시 | 프로토타입 |

**Phase 4 종료 조건:**
- [ ] 코드 노드에서 인프라 노드까지 Neo4j 트래버설 성공
- [ ] MCP로 "BillMapper가 쓰는 DB의 인프라 정보" 질의 가능
- [ ] 역방향 질의 가능

**전제 조건:**
- Ariadne 프로젝트가 Neo4j에 인프라 그래프를 저장하는 상태
- 두 프로젝트가 같은 Neo4j 인스턴스를 사용하거나 Neo4j Fabric으로 연결

---

### Phase 5: 포트폴리오 마감 (2주)

**목표**: 최소 웹 뷰어 + 블로그 + 데모 영상 + 오픈소스 공개

#### Week 12: 웹 뷰어 + 문서

| 작업 | 상세 | 산출물 |
|---|---|---|
| 12-1. 최소 웹 뷰어 | React + Cytoscape.js, Neo4j 서브그래프 시각화 | `web/` |
| 12-2. 드리프트 대시보드 | 데드 코드 후보 / 숨은 경로 목록 | 웹 뷰어 확장 |
| 12-3. README | 설치, 사용법, 아키텍처 다이어그램, 왜 Backstage/Potpie가 아닌가 | `README.md` |
| 12-4. 설정 가이드 | Claude Code MCP 연결, OTel Agent 설정, Docker Compose | `docs/` |

#### Week 13: 블로그 + 데모 + 공개

| 작업 | 상세 | 산출물 |
|---|---|---|
| 13-1. 케이스 스터디 블로그 | dongne-v2 분석 결과 (데드 코드, AOP 경로, 결제 흐름) | 블로그 포스트 |
| 13-2. AI 코딩 에이전트 블로그 | "Claude Code에 코드 GPS를 쥐어주면" | 블로그 포스트 |
| 13-3. 데모 영상 | Claude Code + Theseus MCP 실사용 장면 (2~3분) | YouTube |
| 13-4. 오픈소스 공개 | GitHub public 전환, 라이선스(Apache-2.0), CI/CD | 공개 저장소 |

**Phase 5 종료 조건:**
- [ ] 웹 뷰어에서 dongne-v2 코드 그래프 시각화
- [ ] 블로그 2개 공개
- [ ] 데모 영상 공개
- [ ] GitHub 저장소 공개 + README 완성

---

## 4. 전체 일정 요약

```
Week 1-4:   Phase 1 (정적 그래프 MVP) ← Codex 리뷰 반영, 3주→4주
Week 5-6:   Phase 2 전반 (OTel 수집 + 드리프트 감지)
Week 7-8:   Phase 2 후반 (staging 적용) + Phase 3 전반 (MCP + 임베딩) ← 병행
Week 9:     Phase 3 후반 (Claude Code 통합 테스트)
Week 10-11: Phase 4 (Ariadne 브릿지)
Week 12-13: Phase 5 (포트폴리오 마감)
```

**총 13주 (약 3.5개월)**, Phase 2 후반과 Phase 3 전반의 병행으로 단축.

```
       W1   W2   W3   W4   W5   W6   W7   W8   W9   W10  W11  W12  W13
P1:    ████ ████ ████ ████
P2:                        ████ ████ ▓▓▓▓ ▓▓▓▓              (▓ = staging 데이터 축적)
P3:                                  ████ ████ ████
P4:                                             ████ ████
P5:                                                       ████ ████
```

**주의: Week 1에 dongne-v2 staging 환경 접근 허가 요청 필수** (Phase 2에서 필요, 승인에 2~4주 소요 예상)

---

## 5. 핵심 기술 결정 메모

### Neo4j 스키마 초안

```cypher
// 노드 타입
(:Class {node_id, name, package, file_path, annotations[], type: "controller|service|repository|component|entity"})
(:Method {node_id, name, class_id, visibility, annotations[], signature})
(:Table {node_id, name, source: "mybatis|jpa"})
(:ExternalAPI {node_id, url, name})
(:Config {node_id, key, value, file})

// 엣지 타입
(:Class)-[:HAS_METHOD]->(:Method)
(:Method)-[:CALLS {source: "static|runtime", runtime_count: 0, last_called_at: null}]->(:Method)
(:Method)-[:QUERIES {sql, source: "static|runtime"}]->(:Table)
(:Method)-[:CALLS_API {url}]->(:ExternalAPI)
(:Class)-[:DEPENDS_ON {type: "injection|import"}]->(:Class)
(:Class)-[:USES_CONFIG]->(:Config)

// 브릿지 엣지 (Phase 4)
(:Table)-[:HOSTED_ON]->(:RDS)    // Ariadne 노드
(:ExternalAPI)-[:SERVED_BY]->(:ECS)  // Ariadne 노드
```

### MCP 응답 포맷 (compact)

```json
{
  "query": "결제",
  "nodes": [
    {"id": "abc123", "name": "BillService", "type": "service", "file": "src/main/.../BillService.java", "runtime_calls_30d": 24000},
    {"id": "def456", "name": "BillMapper.insertBill", "type": "mybatis", "table": "bill"}
  ],
  "edges": [
    {"from": "abc123", "to": "def456", "type": "CALLS", "runtime_count": 24000}
  ],
  "drift": [
    {"node": "BillService.legacyRefund", "type": "STATIC_ONLY", "days_inactive": 45}
  ],
  "summary": "결제 처리 흐름: BillService → BillMapper → bill 테이블, ThepayClient 외부 결제 API 호출"
}
```

---

## 6. 의존성 & 선행 조건

| 항목 | 필요 시점 | 상태 |
|---|---|---|
| GitHub 저장소 (Theseus) | Phase 1 시작 | 생성 완료 |
| Neo4j + PostgreSQL 로컬 환경 | Phase 1 Week 1 | 미설정 |
| 장난감 Spring 프로젝트 | Phase 1 Week 1 | 미작성 |
| dongne-v2 소스 접근 | Phase 1 Week 4 | 확인 필요 |
| dongne-v2 staging 환경 접근 | Phase 2 Week 7 | **Week 1에 사전 요청 필수** |
| Ariadne Neo4j 그래��� | Phase 4 Week 9 | Ariadne 진행 상태에 의존 |
| Claude API 키 | Phase 3 Week 8 | 확인 필요 |
| 임베딩 모델 벤치마크 | Phase 3 Week 7 | `all-MiniLM-L6-v2` vs `codebert-base` 비교 필요 |

---

## 7. Codex 리뷰 반영 내역

> 2026-04-13, Codex 독립 리뷰 후 반영

| 피드백 | 조치 |
|---|---|
| Phase 1 기간 과소평가 (tree-sitter 엣지케이스, MyBatis 파서 분량) | **3주 → 4주** 확장. W2에 tree-sitter 안정화 전용 시간 배정, MyBatis/Config 파서를 W3로 분리 |
| PostgreSQL 메타데이터 스키마 누락 | W1에 **PG 스키마 설계 + alembic 마이그레이션** 작업 추가 |
| PG ↔ Neo4j 동기화 전략 부재 | W3에 **동기화 전략** (PG 먼저 기록 → Neo4j 저장 → PG 상태 업데이트) 명시 |
| 증분 파싱 전략 미상세 | W3에 **Git diff 기반 + 파일 해시(SHA-256) PostgreSQL 캐시** 전략 명시 |
| 파싱 실패 시 복구 전략 없음 | W3에 **파일별 try/except, 부분 그래프 생성 보장** 추가 |
| Phase 4 브릿지 노드 타입 미리 예약해야 | W2 Neo4j 스키마에 **Ariadne 브릿지 노드 타입 사전 예약** 추가 |
| 임베딩 모델 `all-MiniLM-L6-v2`가 코드보다 자연어 특화 | W7에 **`codebert-base` 벤치마크 후 결정** 단계 추가 |
| 드리프트 감지 정확도 검증 방법 없음 | W6에 **골든 데이터셋**(의도적 데드 코드 + AOP 경로 포함) 기반 검증 추가 |
| 단위 테스트 전략 부재 | W1에 **픽스처 기반 단위 테스트 프레임** 추가, 모든 파서 변경의 리그레션 방지 |
| MCP 에러 핸들링 / 안정성 | W9에 **타임아웃, Neo4j 다운 시 graceful 에러, 응답 사이즈 cap** 추가 |
| staging 접근 허가 일정 계획에 없음 | **Week 1에 사전 요청** 명시 + 의존성 표 업데이트 |
| 총 일정 | **12주 → 13주** (Phase 1 +1주) |
