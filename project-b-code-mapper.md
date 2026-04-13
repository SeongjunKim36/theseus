# Theseus — 코드 토폴로지 MCP 서버 기획서

> 상태: Phase 0 (논제 고정) · 최초 작성: 2026-04-10 · 피벗: 2026-04-13 · 관련 문서: [Ariadne - AWS 인프라 맵퍼](./project-a-infra-mapper.md)

## 이름의 유래

그리스 신화에서 테세우스(Theseus)는 아리아드네가 쥐어준 실타래를 따라 미궁을 헤쳐나간 영웅이다. 이 프로젝트는 복잡하게 얽힌 코드베이스라는 미궁을, 정적 분석과 런타임 데이터라는 실을 따라 탐험한다. 자매 프로젝트 [Ariadne](./project-a-infra-mapper.md)가 외부 인프라 세계를 다룬다면, Theseus는 코드 내부 세계를 다룬다.

## 한 줄 요약

**정적 코드 분석 + 런타임 트레이스 + LLM 의미 이해**를 하나의 그래프로 묶어, **사람과 AI 에이전트 모두에게** 코드베이스 컨텍스트를 제공하는 **MCP 기반 코드 토폴로지 서버**.

## 왜 이걸 만드나 (문제 정의)

### 사람의 문제

- 레거시 Spring Boot 프로젝트(dongne-v2 같은)에서 "이 메서드 고치면 뭐가 깨져?"에 답하려면 grep + 읽기 + 사람에게 묻기를 반복해야 한다
- Spring의 AOP/프록시/리플렉션 때문에 **정적 분석만으로는 실제 호출 관계를 알 수 없다**
- APM(런타임 관찰) 도구는 **코드의 의도**를 모른다 ("이게 왜 이렇게 설계됐나")
- 코드베이스가 수백 개 클래스를 넘으면 "결제 흐름이 어떻게 생겼어?"에 단일한 답이 없다
- 온보딩할 때 "시니어가 설명해줘야만" 이해되는 지식이 너무 많다

### AI 에이전트의 문제 (피벗 동기)

- AI 코딩 도구(Claude Code, Cursor, Aider 등)는 매 세션마다 `grep`/`find`/`Read`를 반복하여 **코드베이스를 처음부터 다시 이해한다**
- 컨텍스트 윈도우의 40~60%를 "코드 탐색"에 소모하고, 실제 "코드 작성"에 쓸 수 있는 토큰이 줄어든다
- AI는 정적 텍스트만 보므로 **런타임에 실제로 타는 경로**를 모른다 — AOP/프록시 뒤의 호출을 놓침
- AI가 변경의 영향 범위를 추측할 수밖에 없어 **사이드 이펙트를 자주 만든다**
- 여러 AI 도구를 쓰더라도 각각이 독립적으로 같은 탐색을 반복한다

### 핵심 인사이트

> AI 코딩 에이전트에게 매번 지도를 처음부터 그리게 하지 말고, **GPS를 쥐어주자.**
> 구조화된 코드 토폴로지 스냅샷 하나면, grep 10회를 MCP 1회 질의로 대체할 수 있다.

## 타겟 사용자 & 유스케이스

### 1차 타겟: AI 코딩 에이전트 (MCP 클라이언트)

- **Claude Code / Cursor / Aider**: MCP 서버로 연결하여 코드베이스 컨텍스트를 즉시 획득
  - "결제 관련 서브그래프 + 영향 범위 + 런타임 빈도" → 구조화된 JSON 응답
  - 변경 전 blast radius 확인 → 사이드 이펙트 방지
  - 데드 코드/라이브 코드 구분 → 불필요한 코드 건드림 방지
- **CI/CD 파이프라인**: PR 자동 영향 범위 분석 → 코멘트 작성

### 2차 타겟: 사람 개발자 (웹 뷰어)

- **기존 코드에 기능을 추가하려는 개발자**: "이거 건드리면 뭐가 영향받아?"
- **온보딩 중인 신입**: 자연어로 코드베이스 탐색
- **리뷰어**: PR의 영향 범위 자동 리포트
- **아키텍트**: 데드 코드 / 숨은 의존성 발견

## 성공 지표

이 6개에 답할 수 있으면 프로젝트 완료로 간주한다.

### 사람 대상

1. **"결제 관련 모든 코드/테이블/외부 API 보여줘"** → 자연어 질의로 서브그래프 반환
2. **"이 PR이 바꾸는 메서드의 영향 범위는?"** → CI에 꽂혀 PR 코멘트 자동 작성
3. **"정적으로는 A→B 호출인데, 런타임에 30일간 아무도 안 탔어"** → 데드 코드 후보 리포트

### AI 에이전트 대상

4. **AI 에이전트가 grep 10회 할 것을 MCP 1회 질의로 해결** → 코드 탐색 라운드트립 80% 감소
5. **AI 코딩 세션에서 "코드 이해" 토큰 소모 50% 감소** → 같은 컨텍스트 윈도우로 더 복잡한 작업 가능

### 시너지 대상

6. **Ariadne 연결 시 "BillService가 쓰는 DB가 죽으면 영향 범위는?"** → 코드 ~ 인프라 전경로 추적

## 비범위 (Non-goals)

- **IDE 플러그인 아님** — MCP 프로토콜로 모든 AI 도구에 범용 연결, 특정 IDE 종속 X
- **코드 포맷터/린터 아님** — 스타일 검사 안 함
- **리팩토링 자동화 아님** — 읽기/이해/컨텍스트 제공 도구이지 코드를 고치지 않음
- **모든 언어 지원 아님 (최소 1년간)** — Java/Spring만. Kotlin/TypeScript는 나중
- **커밋 히스토리 분석 도구 아님** — git log는 git이 잘 함
- **정적 분석 파이프라인 재발명 아님** — Potpie AI 오픈소스를 기반으로, 없는 것만 만든다

## 기존 툴 대비 차별점

| 툴 | 제공하는 것 | AI 에이전트 지원 | 런타임 데이터 | 이 프로젝트의 차이 |
|---|---|---|---|---|
| Backstage | 사람이 YAML로 입력한 카탈로그 | X | X | 코드에서 자동 추출 + MCP |
| Datadog APM | 런타임 트레이스 | X | O | 코드 의도 + 정적 구조 결합 |
| IntelliJ "Find Usages" | IDE 내 참조 검색 | X | X | 런타임 데이터 + AI 접근성 |
| Sourcegraph 7.0 | Code Graph + Deep Search + MCP | O (MCP) | X | **런타임 가중치 + 드리프트 감지** |
| Potpie AI | 정적 AST → Neo4j + AI 에이전트 | 자체 에이전트만 | X | **런타임 레이어 + 범용 MCP + 인프라 브릿지** |
| CodeScene | 히트맵 + 기술 부채 | X | 부분적 | 토폴로지 + MCP + 오픈소스 |
| Aider repo-map | tree-sitter 함수 시그니처 | 자체만 | X | **그래프 구조 + 런타임 + 범용 MCP** |

### 한 줄 차별점

**Potpie AI(정적 분석)를 기반으로, 런타임 드리프트 감지 + MCP 서버 + Ariadne 인프라 브릿지를 얹은 "AI 에이전트를 위한 코드 GPS".**

## 전략: Build vs. Extend

### 왜 처음부터 만들지 않는가

2026년 4월 기준, 정적 분석 → 그래프 DB 파이프라인은 이미 해결된 문제다:
- **Potpie AI**: 오픈소스, Neo4j 기반, AST → 지식 그래프, 2026년 3월 $2.2M 투자
- **Sourcegraph 7.0**: Code Graph(시맨틱, 심볼, 참조, 의존성 트리) 출시

이걸 혼자 6~7개월 재구현하면 좋은 엔지니어링 판단이 아니다.

### 무엇을 Extend하는가

Potpie AI 오픈소스를 기반(포크)으로, **아무도 아직 안 한 3가지**만 만든다:

1. **런타임 드리프트 감지** — OpenTelemetry 데이터로 정적 그래프에 "실제 호출 빈도" 가중치 부여 + "코드엔 있는데 안 탐" / "코드엔 없는데 불림" 감지
2. **범용 MCP 서버** — 어떤 AI 코딩 도구든 연결하여 구조화된 토폴로지 스냅샷 질의 가능
3. **Ariadne 인프라 브릿지** — 코드 그래프(Theseus) ↔ 인프라 그래프(Ariadne) Neo4j 엣지 연결

## 기술 스택

- **기반**: Potpie AI 오픈소스 포크 (정적 분석 + Neo4j 그래프)
- **런타임 수집**: OpenTelemetry Java Agent
- **그래프 저장소**: Neo4j (Potpie 기본 + Project A 공유)
- **MCP 서버**: Python (Potpie 런타임과 동일 스택) 또는 TypeScript
- **프론트**: React + Cytoscape.js + Tailwind (Phase 5에서 최소한으로)
- **AI**: Claude API (claude-sonnet-4-6), 자연어 질의 → 서브그래프 매핑

## 아키텍처

```
┌─────────────────────────────────────────────────────┐
│                    Theseus 서버                       │
│                                                       │
│  ┌──────────────┐  ┌──────────────┐  ┌────────────┐ │
│  │ 정적 분석     │  │ 런타임 수집   │  │ LLM 의미층  │ │
│  │ (Potpie 기반) │  │ (OTel Agent) │  │ (Claude)   │ │
│  └──────┬───────┘  └──────┬───────┘  └─────┬──────┘ │
│         │                  │                │        │
│         └──────────┬───────┘                │        │
│                    ▼                        │        │
│           ┌──────────────┐                  │        │
│           │   Neo4j      │◄─────────────────┘        │
│           │ (통합 그래프)  │                           │
│           └──────┬───────┘                           │
│                  │                                    │
│    ┌─────────────┼─────────────┐                     │
│    ▼             ▼             ▼                     │
│ ┌──────┐  ┌──────────┐  ┌──────────┐               │
│ │ MCP  │  │ REST API │  │ Web UI   │               │
│ │서버   │  │          │  │(Cytoscape)│               │
│ └──┬───┘  └────┬─────┘  └────┬─────┘               │
└────┼───────────┼──────────────┼──────────────────────┘
     │           │              │
     ▼           ▼              ▼
  AI 도구      CI/CD         사람 개발자
(Claude Code  (GitHub       (브라우저)
 Cursor       Actions)
 Aider)
                    │
                    ▼
            ┌──────────────┐
            │  Ariadne     │
            │ (인프라 그래프) │
            │  Neo4j       │
            └──────────────┘
```

### 삼중 보완 구조

```
정적 분석 (코드 구조)  ←→  런타임 트레이스 (실제 행동)
         ↘                    ↙
           LLM 의미층 (의도 이해)
```

- **정적 분석 한계**(AOP/리플렉션) → 런타임이 보완
- **런타임 한계**(의도 파악 불가) → LLM이 보완
- **LLM 한계**(환각) → 정적/런타임 데이터가 근거(grounding) 제공

## MCP 서버 인터페이스 설계 (초안)

### 제공할 MCP Tools

```
theseus/get_subgraph
  - 입력: 자연어 질의 ("결제 관련 코드") 또는 진입점 ("BillService")
  - 출력: 관련 노드/엣지 서브그래프 (JSON)
  - 포함: 런타임 호출 빈도, 마지막 호출 시각

theseus/blast_radius
  - 입력: 파일 경로 또는 메서드명
  - 출력: 영향 받는 다운스트림 노드 목록 + 심각도

theseus/drift_report
  - 입력: 없음 (전체) 또는 모듈명
  - 출력: 정적 vs 런타임 불일치 목록
    - "코드엔 있는데 30일간 안 탐" (데드 코드 후보)
    - "코드엔 없는데 불림" (리플렉션/AOP 숨은 경로)

theseus/snapshot
  - 입력: 스코프 (전체 / 모듈 / 특정 서비스)
  - 출력: 압축된 토폴로지 스냅샷 (AI 컨텍스트 주입용)

theseus/ariadne_bridge  (Phase 4)
  - 입력: 코드 노드 (e.g., "BillMapper")
  - 출력: 연결된 인프라 노드 (RDS, S3, VPC 등)
```

### 사용 시나리오 예시

```
[사용자 → Claude Code]
"결제 로직에 할인 기능 추가해줘"

[Claude Code → Theseus MCP]
theseus/get_subgraph { query: "결제" }
theseus/blast_radius { target: "BillService.createBill" }

[Theseus → Claude Code]
{
  "subgraph": {
    "nodes": [
      { "name": "BillService.createBill", "type": "method", "runtime_calls_30d": 24000 },
      { "name": "BillMapper.insertBill", "type": "mybatis", "table": "bill" },
      { "name": "ThepayClient.requestPayment", "type": "external_api" }
    ],
    "edges": [...],
    "dead_paths": ["BillService.legacyRefund → 30일간 호출 0"]
  },
  "blast_radius": {
    "downstream": ["NotificationService", "ReceiptService"],
    "severity": "high",
    "reason": "결제 테이블 스키마 변경 시 3개 서비스 영향"
  }
}

[Claude Code]
→ 즉시 코드 작성 시작 (grep/Read 없이 구조 파악 완료)
→ legacyRefund는 건드리지 않음 (데드 코드 정보 활용)
→ NotificationService 영향 고려하여 작성
```

## 로드맵 (13주, 약 3.5개월)

> 상세 개발 계획: [DEVELOPMENT_PLAN.md](./DEVELOPMENT_PLAN.md)

### Phase 1: 정적 그래프 MVP (4주)

- Potpie AI **참고(Reference)**, 포크 아님 — 핵심 파싱 패턴만 차용, SaaS 의존성 배제
- tree-sitter + Spring 특화 파서(어노테이션, MyBatis XML, application.yml)
- Neo4j 그래프 빌더 + 증분 파싱(Git diff + 파일 해시)
- dongne-v2 적용 → 파싱 정확도 검증
- **종료 조건**: dongne-v2의 의미 있는 코드 그래프가 Neo4j에 생성됨

### Phase 2: 런타임 드리프트 레이어 (3~4주) ← 핵심 차별점

- OpenTelemetry Java Agent로 dongne-v2 staging 계측
- HTTP in/out, DB 쿼리, 외부 API 호출, 이벤트 캡처
- 정적 그래프에 "실제 호출 빈도" 엣지 가중치 부여
- **드리프트 감지 엔진**: 
  - "정적엔 있는데 안 탐" → 데드 코드 후보
  - "정적엔 없는데 실제로 불림" → 리플렉션/AOP 숨은 경로
- **종료 조건**: 드리프트 리포트에서 실제 데드 코드 후보 3개 이상 발견

### Phase 3: MCP 서버 + AI 의미층 (3주, Phase 2와 병행) ← 포트폴리오 하이라이트

- FastMCP 서버 구현 (5개 tool: get_subgraph, blast_radius, drift_report, snapshot, ariadne_bridge)
- 코드 임베딩 (`all-MiniLM-L6-v2` vs `codebert-base` 벤치마크 후 결정)
- 자연어 질의 → 서브그래프 반환 (임베딩 유사도 + 그래프 트래버설)
- Claude Code 통합 테스트 + 응답 최적화
- **종료 조건**: Claude Code에서 "결제 관련 코드 보여줘" 질의 → 정확한 서브그래프 반환

### Phase 4: Ariadne 브릿지 (2주) ← 킬러 시너지

- Theseus Neo4j ↔ Ariadne Neo4j 엣지 연결
- `theseus/ariadne_bridge` MCP tool 구현
- "BillMapper가 쓰는 DB → RDS → VPC → 같은 RDS 쓰는 다른 서비스" 전경로
- **종료 조건**: 코드 노드에서 인프라 노드까지 그래프 트래버설 성공

### Phase 5: 포트폴리오 마감 (2주)

- 최소한의 웹 뷰어 (Cytoscape.js) — MCP가 주 인터페이스이므로 가벼우게
- 케이스 스터디 블로그:
  - "dongne-v2에 Theseus 돌렸더니 찾은 것들 N가지"
  - "AI 코딩 에이전트에 코드 GPS를 쥐어줬더니"
  - "코드에서 인프라까지 — Theseus + Ariadne 시너지"
- 데모 영상: Claude Code + Theseus MCP 실제 사용 장면
- 오픈소스 공개 + README

## 리스크와 대응

### 기존 리스크

1. **스코프 크립이 최대 위협** — "Backstage 재발명"으로 새면 끝 없음
   - 대응: Potpie 기반이므로 정적 분석 파이프라인은 건드리지 않음. 런타임/MCP/브릿지만 만든다
2. **Spring 리플렉션/AOP/프록시는 정적 분석 한계** — `@Transactional`, `@Async`, `@EventListener` 등
   - 대응: Phase 2 런타임 데이터로 보완. 드리프트 감지가 오히려 이 한계를 feature로 전환
3. **LLM 임베딩 비용** — 대형 코드베이스 풀 임베딩
   - 대응: Potpie의 증분 업데이트 활용 + 로컬 임베딩 모델(nomic-embed 등) 대안 준비
4. **Java Agent 오버헤드** — CPU 2~5%, 메모리 50~100MB 추가
   - 대응: 샘플링 전략(50% 감소 가능), staging에 먼저 적용
5. **파싱 엣지 케이스** — Potpie가 Spring/MyBatis를 완벽히 처리하지 못할 수 있음
   - 대응: 장난감 프로젝트 먼저 검증, 필요시 파서 확장 기여

### 추가 식별 리스크 (피벗 후)

6. **Sourcegraph 7.0 / Potpie AI의 빠른 진화** — 유사 기능이 곧 추가될 수 있음
   - 대응: 런타임 드리프트 감지 + Ariadne 브릿지는 이들이 쉽게 따라올 수 없는 영역. 속도가 중요
7. **1인 개발 병목** — 런타임 + MCP + 브릿지 + 웹 UI를 동시에
   - 대응: 웹 UI를 Phase 5로 미루고, MCP 서버(AI 인터페이스)를 주 인터페이스로 먼저 개발
8. **dongne-v2 staging 환경 의존** — Phase 2가 회사 인프라에 종속
   - 대응: 장난감 프로젝트에 OTel 먼저 검증 후 staging 적용. 환경 접근 사전 확보
9. **MCP 프로토콜 안정성** — MCP가 아직 초기 표준, 스펙 변경 가능
   - 대응: 핵심 로직은 REST API로도 노출, MCP는 얇은 어댑터 레이어로 분리
10. **Potpie AI 라이선스/포크 관리** — upstream 변경 추적 부담
    - 대응: 최소한의 포크 (런타임 레이어만 추가), upstream과 충돌 최소화

## 킬러 시너지 (Project A와 결합 시)

두 프로젝트 모두 Neo4j 기반이므로 하나의 엣지로 연결:

```
[코드: BillService.createBill]
        ↓
[코드: BillMapper → bill 테이블]
        ↓ (← Theseus ↔ Ariadne 브릿지)
[인프라: RDS dongne-prod-db]
        ↓
[인프라: 같은 RDS 쓰는 다른 ECS 태스크 / 백업 S3 / 속한 VPC]
```

질문 예: **"BillService가 쓰는 DB가 죽으면 영향 범위는?"** — 코드부터 하드웨어까지 전 경로 추적.

AI 에이전트도 MCP로 같은 질문 가능:
```
theseus/ariadne_bridge { source: "BillMapper" }
→ RDS dongne-prod-db → ECS task-api, task-batch → S3 backup-bucket
```

이 조합을 깔끔하게 해낸 오픈소스는 현재 없음. **Theseus + Ariadne = 코드에서 인프라까지 관통하는 유일한 오픈소스 토폴로지 플랫폼.**

## 미정 사항 (결정 필요)

- [x] 프로젝트 이름: **Theseus** (2026-04-10 확정)
- [x] 전략: Potpie AI 기반 Extend (2026-04-13 확정)
- [x] 주 인터페이스: MCP 서버 (2026-04-13 확정)
- [ ] 저장소 호스팅 (GitHub 계정)
- [ ] Potpie AI 포크 vs 플러그인으로 확장 (소스 분석 후 결정)
- [ ] MCP 서버 구현 언어 (Python vs TypeScript)
- [ ] Phase 1 검증용 장난감 Spring 프로젝트 선정
- [ ] Project A(Ariadne) 끝난 후 시작 vs 병행
- [ ] Phase 2 계측 범위: dongne-v2 전체 vs 특정 모듈만
