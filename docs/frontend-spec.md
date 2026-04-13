# Frontend Specification — Theseus Web Viewer

> Implemented in Phase 5 (Week 12) · Since the MCP server is the primary interface, **focus on minimum features**

---

## Tech Stack

| Technology | Version | Rationale |
|---|---|---|
| React | 18+ | Component-based, ecosystem |
| TypeScript | 5+ | Type safety |
| Cytoscape.js | 3.x | De facto standard for graph visualization, Neo4j-friendly |
| Tailwind CSS | 3+ | Rapid styling, utility-first |
| Vite | 5+ | Fast builds |
| React Query | 5+ | Server state management |

---

## Page Structure

```
/                          → Project list
/project/:id               → Graph viewer (main)
/project/:id/drift          → Drift dashboard
/project/:id/stats          → Project statistics
```

---

## Screen 1: Graph Viewer (Main)

### Wireframe

```
┌─────────────────────────────────────────────────────────┐
│  🧭 Theseus   [Select Project ▾]   [Search: _________ 🔍]│
├──────────┬──────────────────────────────────────────────┤
│          │                                              │
│ Filters  │         Cytoscape.js Graph Canvas            │
│          │                                              │
│ Layer    │    [Controller] ──→ [Service] ──→ [Repo]     │
│ □ Controller │              │                           │
│ □ Service    │     [Service] ──→ [Table]                │
│ □ Repository │              │                           │
│ □ Entity     │   [ExternalAPI] ←── [Service]            │
│          │                                              │
│ Source   │                                              │
│ □ Static │                                              │
│ □ Runtime│                                              │
│ □ Both   │                                              │
│          │                                              │
│ Drift    │                                              │
│ □ Dead Code  │                                          │
│ □ Hidden Path│                                          │
│          │                                              │
├──────────┴──────────────────────────────────────────────┤
│ Node Details                                             │
│ BillService (@Service)                                   │
│ File: src/.../BillService.java:23                        │
│ Methods: createBill, getBill, legacyRefund(⚠️ dead)      │
│ Runtime 30d: 24,000 calls                                │
│ Dependencies: BillMapper, NotificationService, ThepayClient│
└─────────────────────────────────────────────────────────┘
```

### Component Structure

```
<App>
├── <Header>                    // Project selection, search
├── <MainLayout>
│   ├── <FilterPanel>           // Left: Layer/Source/Drift filters
│   ├── <GraphCanvas>           // Center: Cytoscape.js
│   │   ├── useCytoscape()      // Cytoscape instance management
│   │   ├── useGraphData()      // API → Cytoscape data transformation
│   │   └── useGraphLayout()    // Layout algorithm
│   └── <NodeDetail>            // Bottom: Selected node details
└── <Footer>
```

### Cytoscape.js Node Styles

```typescript
const STYLE: cytoscape.Stylesheet[] = [
  // Node defaults
  {
    selector: 'node',
    style: {
      'label': 'data(name)',
      'text-valign': 'center',
      'font-size': '12px',
      'width': 'mapData(runtime_calls, 0, 10000, 30, 80)',  // Size proportional to call volume
      'height': 'mapData(runtime_calls, 0, 10000, 30, 80)',
    }
  },
  // Color by layer
  {
    selector: 'node[type="controller"]',
    style: { 'background-color': '#3B82F6' }  // blue
  },
  {
    selector: 'node[type="service"]',
    style: { 'background-color': '#10B981' }  // green
  },
  {
    selector: 'node[type="repository"]',
    style: { 'background-color': '#F59E0B' }  // amber
  },
  {
    selector: 'node[type="table"]',
    style: { 'background-color': '#6366F1', 'shape': 'rectangle' }  // indigo, rectangle
  },
  {
    selector: 'node[type="external_api"]',
    style: { 'background-color': '#EF4444', 'shape': 'diamond' }  // red, diamond
  },
  // Dead code highlight
  {
    selector: 'node[drift_type="STATIC_ONLY"]',
    style: { 'border-width': 3, 'border-color': '#EF4444', 'border-style': 'dashed' }
  },
  // Edges
  {
    selector: 'edge',
    style: {
      'curve-style': 'bezier',
      'target-arrow-shape': 'triangle',
      'width': 'mapData(runtime_count, 0, 10000, 1, 5)',  // Thickness proportional to call volume
    }
  },
  // Runtime-only edges (hidden paths)
  {
    selector: 'edge[source_type="runtime"]',
    style: { 'line-style': 'dashed', 'line-color': '#EF4444' }
  },
];
```

### Layout Algorithm

```typescript
const LAYOUT_OPTIONS = {
  name: 'dagre',           // Directed graph layout
  rankDir: 'TB',           // Top → Bottom (Controller → Service → Repo)
  rankSep: 80,
  nodeSep: 60,
  animate: true,
  animationDuration: 300,
};
```

### Interactions

| Action | Result |
|---|---|
| Node click | Show detailed info in bottom panel |
| Node double-click | Expand subgraph centered on that node |
| Edge hover | Tooltip showing call frequency, source type |
| Search input | Highlight matching node + move focus |
| Filter checkbox | Real-time node/edge filtering |
| Mouse wheel | Zoom in/out |
| Drag | Panning |

---

## Screen 2: Drift Dashboard

### Wireframe

```
┌─────────────────────────────────────────────────────────┐
│  🧭 Theseus > dongne-v2 > Drift Report                  │
├─────────────────────────────────────────────────────────┤
│                                                          │
│  Summary                                                 │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐               │
│  │ Dead Code │  │ Hidden   │  │ Coverage │               │
│  │    3     │  │ Paths: 2 │  │  87.2%   │               │
│  └──────────┘  └──────────┘  └──────────┘               │
│                                                          │
│  Dead Code Candidates (present in static, unused at runtime)│
│  ┌───────────────────────────────────────────────┐      │
│  │ ⚠ BillService.legacyRefund()                   │      │
│  │   File: src/.../BillService.java:145            │      │
│  │   Unused for: 45 days | Severity: medium        │      │
│  ├───────────────────────────────────────────────┤      │
│  │ ⚠ UserService.migrateOldProfile()              │      │
│  │   File: src/.../UserService.java:89             │      │
│  │   Unused for: 90 days | Severity: high          │      │
│  └───────────────────────────────────────────────┘      │
│                                                          │
│  Hidden Paths (runtime-only, not reflected in static analysis)│
│  ┌───────────────────────────────────────────────┐      │
│  │ 🔍 TransactionInterceptor → BillService        │      │
│  │   Mechanism: AOP Proxy | Calls: 24,000/30d     │      │
│  ├───────────────────────────────────────────────┤      │
│  │ 🔍 EventListenerProxy → NotificationService    │      │
│  │   Mechanism: EventListener | Calls: 1,200/30d  │      │
│  └───────────────────────────────────────────────┘      │
│                                                          │
└─────────────────────────────────────────────────────────┘
```

### Components

```
<DriftDashboard>
├── <DriftSummaryCards>        // Dead code count, hidden path count, coverage
├── <DeadCodeTable>            // Dead code candidate list (sortable/filterable)
│   └── <DeadCodeRow>          // Individual item → click to navigate to graph viewer
├── <HiddenPathTable>          // Hidden path list
│   └── <HiddenPathRow>
└── <CoverageChart>            // Static vs runtime edge coverage ratio chart
```

---

## Screen 3: Project Statistics

```
┌─────────────────────────────────────────┐
│  Project: dongne-v2                      │
│  Last parsed: 2026-04-20 15:30           │
│  Status: ready                           │
│                                          │
│  Graph Statistics                        │
│  ├── Classes: 142                        │
│  ├── Methods: 890                        │
│  ├── Tables: 35                          │
│  ├── External APIs: 8                    │
│  ├── Static edges: 450                   │
│  └── Runtime edges: 520                  │
│                                          │
│  Parsing Quality                         │
│  ├── Successful files: 148/152 (97.4%)   │
│  ├── Failed files: 4 (view details)      │
│  └── Last incremental parse: 12 files (0.3s)│
│                                          │
│  Layer Distribution                      │
│  Controller ████████ 12                  │
│  Service    ██████████████████ 38        │
│  Repository ████████████ 24              │
│  Entity     ██████████████████████ 45    │
│  Other      ████████ 23                  │
└─────────────────────────────────────────┘
```

---

## API Endpoints (Frontend → Backend)

```
GET  /api/projects                    → Project list
GET  /api/projects/:id/graph          → Full graph (Cytoscape format)
GET  /api/projects/:id/graph?query=   → Subgraph search
GET  /api/projects/:id/graph/:nodeId  → Node details + neighbors
GET  /api/projects/:id/drift          → Drift report
GET  /api/projects/:id/stats          → Project statistics
POST /api/projects/:id/parse          → Trigger parsing
```

### Graph API Response (Cytoscape Format)

```typescript
interface GraphResponse {
  elements: {
    nodes: CytoscapeNode[];
    edges: CytoscapeEdge[];
  };
  metadata: {
    total_nodes: number;
    total_edges: number;
    query_time_ms: number;
  };
}

interface CytoscapeNode {
  data: {
    id: string;
    name: string;
    type: 'controller' | 'service' | 'repository' | 'table' | 'external_api' | 'config';
    file_path?: string;
    runtime_calls?: number;
    drift_type?: 'STATIC_ONLY' | 'RUNTIME_ONLY' | null;
    annotations?: string[];
  };
}

interface CytoscapeEdge {
  data: {
    id: string;
    source: string;
    target: string;
    type: 'CALLS' | 'DEPENDS_ON' | 'QUERIES' | 'CALLS_API';
    source_type: 'static' | 'runtime' | 'both';
    runtime_count?: number;
  };
}
```

---

## Out of Scope (Not included in this Phase)

- User authentication/login
- Real-time WebSocket updates
- Graph editing/modification features
- Mobile responsive design (desktop only)
- Dark mode (first version)
- Internationalization (Korean only)
