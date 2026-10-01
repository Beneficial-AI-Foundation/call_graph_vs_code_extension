# Pre-computed Call Graph Index Design

> **Goal**: Achieve instant O(1) subgraph lookup instead of O(n) full graph computation when clicking on a function.

## Key Architectural Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| **Index Format** | D3 Graph JSON from scip-callgraph | Already has `dependencies` + `dependents` arrays for O(1) lookup |
| **Build Tool** | `scip-callgraph pipeline --skip-similar-lemmas` | Runs verus-analyzer, builds graph, adds verification status |
| **Spec Parsing** | verus_syn (in scip-callgraph) | Parses `requires`/`ensures` clauses for spec change detection |
| **Enrichments** | Verification status via `cargo verus verify` | Shows verified/failed/unverified for each function |
| **Graph Updates** | verus-analyzer LSP (not tree-sitter) | LSP provides semantic symbol resolution; tree-sitter only provides syntax |
| **Verification Updates** | Full `cargo verus verify` | ~12s for full project; per-function has ~6s overhead, not worth the complexity |
| **Symbol Resolution** | verus-analyzer LSP `executeDefinitionProvider` | Accurate resolution of `helper()` → full SCIP symbol |

---

## Table of Contents

1. [Problem Statement](#problem-statement)
2. [Proposed Solution](#proposed-solution)
3. [Index Structure](#index-structure)
4. [Enrichments from Pipeline](#enrichments-from-pipeline)
5. [Query-time Subgraph Extraction](#query-time-subgraph-extraction)
6. [Updates: Just Re-run the Pipeline](#updates-just-re-run-the-pipeline)
7. [Implementation Plan](#implementation-plan)
8. [Open Questions](#open-questions)

---

## Problem Statement

### Current Architecture

```
User clicks on function f
       │
       ▼
┌──────────────────────────────┐
│ Load index_scip.json (full)  │  ← O(file_size) ~5-50MB
└──────────────────────────────┘
       │
       ▼
┌──────────────────────────────┐
│ Build entire call graph      │  ← O(n) where n = all functions
│ - Parse all documents        │
│ - Build all edges            │
│ - Extract all bodies         │
└──────────────────────────────┘
       │
       ▼
┌──────────────────────────────┐
│ Generate subgraph for f      │  ← O(depth × degree)
└──────────────────────────────┘
       │
       ▼
┌──────────────────────────────┐
│ Convert to DOT → SVG         │
└──────────────────────────────┘
```

**Pain Points**:
- For a project with 10,000 functions, we rebuild the entire graph every click
- SCIP JSON parsing is expensive (JSON parse + symbol resolution)
- Body extraction reads source files repeatedly
- Typical latency: **5-15 seconds** per query

### Desired State

```
User clicks on function f
       │
       ▼
┌──────────────────────────────┐
│ Lookup f in pre-loaded index │  ← O(1) HashMap lookup
└──────────────────────────────┘
       │
       ▼
┌──────────────────────────────┐
│ BFS traverse dependencies/   │  ← O(depth × avg_degree)
│ dependents to depth d        │     Typically < 100 nodes
└──────────────────────────────┘
       │
       ▼
┌──────────────────────────────┐
│ Render subgraph              │  ← Instant
└──────────────────────────────┘
```

**Expected latency: 10-50ms** (100-1000x improvement)

---

## Proposed Solution

### Overview

We use **scip-callgraph**'s `pipeline` command to generate an enriched call graph index. This format contains:

1. **Forward edges** (`dependencies`): What each function calls
2. **Backward edges** (`dependents`): What calls each function
3. **Verification status**: `verified`, `failed`, or `unverified` (from running Verus)

This is essentially a **bidirectional adjacency list** with verification metadata that allows O(1) neighbor lookup.

### Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     BUILD TIME (once, or on demand)                     │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌─────────────┐                                                        │
│  │ Source Code │                                                        │
│  │   *.rs      │                                                        │
│  └──────┬──────┘                                                        │
│         │                                                               │
│         ▼                                                               │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │  scip-callgraph pipeline /path/to/project --skip-similar-lemmas  │  │
│  │                                                                   │  │
│  │  Internally runs:                                                 │  │
│  │  1. verus-analyzer scip .           → index.scip                 │  │
│  │  2. scip print --json               → index.scip.json            │  │
│  │  3. Build call graph with atoms     → dependencies/dependents    │  │
│  │  4. Run Verus verification          → verification_status        │  │
│  │  5. Export enriched D3 graph        → graph.json                 │  │
│  └──────────────────────────────────────────────────────────────────┘  │
│                                                        │               │
│                                                        ▼               │
│                                          ┌──────────────────────────┐  │
│                                          │  call_graph_index.json   │  │
│                                          │  (fully enriched index)  │  │
│                                          └──────────────────────────┘  │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────┐
│                        QUERY TIME (every click)                         │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌───────────────┐    ┌──────────────────┐    ┌────────────────────┐   │
│  │ User clicks   │───▶│ IndexLoader      │───▶│ SubgraphExtractor  │   │
│  │ on function f │    │ (cached in mem)  │    │ BFS traversal      │   │
│  └───────────────┘    └──────────────────┘    └────────────────────┘   │
│                                                        │               │
│                                                        ▼               │
│                                          ┌──────────────────────────┐  │
│                                          │  WebviewRenderer         │  │
│                                          │  (D3.js or DOT→SVG)      │  │
│                                          └──────────────────────────┘  │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────┐
│                  INCREMENTAL UPDATE (on file save)                      │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌───────────────┐    ┌──────────────────┐    ┌────────────────────┐   │
│  │ File saved    │───▶│ verus-analyzer   │───▶│ DeltaCalculator    │   │
│  │ (*.rs)        │    │ LSP queries      │    │ Diff old vs new    │   │
│  └───────────────┘    │ (already running)│    └────────────────────┘   │
│                       └──────────────────┘             │               │
│                                                        ▼               │
│  Uses:                                    ┌──────────────────────────┐  │
│  • executeDocumentSymbolProvider          │  Apply delta to index    │  │
│  • executeDefinitionProvider              │  (in-memory, then disk)  │  │
│                                           └──────────────────────────┘  │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Index Structure

### D3 Graph JSON Format (from scip-callgraph)

```json
{
  "nodes": [
    {
      "id": "rust-analyzer cargo my_crate 0.1.0 module/MyStruct#my_function().",
      "display_name": "my_function",
      "symbol": "rust-analyzer cargo my_crate 0.1.0 module/MyStruct#my_function().",
      "relative_path": "src/module.rs",
      "file_name": "module.rs",
      "parent_folder": "src",
      "start_line": 42,
      "end_line": 58,
      "is_libsignal": false,
      "dependencies": [
        "rust-analyzer cargo my_crate 0.1.0 utils#helper().",
        "rust-analyzer cargo my_crate 0.1.0 module/MyStruct#inner_call()."
      ],
      "dependents": [
        "rust-analyzer cargo my_crate 0.1.0 main#entry_point()."
      ],
      "mode": "exec"
    }
    // ... more nodes
  ],
  "links": [
    {
      "source": "...",
      "target": "...",
      "type": "inner"  // or "precondition", "postcondition"
    }
  ],
  "metadata": {
    "total_nodes": 1234,
    "total_edges": 5678,
    "project_root": "/home/user/project",
    "generated_at": "2025-12-26T10:00:00Z",
    "github_url": "https://github.com/org/repo"
  }
}
```

### In-Memory Index (TypeScript)

```typescript
interface D3Node {
  id: string;
  display_name: string;
  symbol: string;
  relative_path: string;
  file_name: string;
  parent_folder: string;
  start_line?: number;
  end_line?: number;
  is_libsignal: boolean;
  dependencies: string[];   // IDs of functions this calls
  dependents: string[];     // IDs of functions that call this
  mode: 'exec' | 'proof' | 'spec';
  
  // Enrichment from pipeline
  verification_status?: 'verified' | 'failed' | 'unverified';
}

interface CallGraphIndex {
  // Primary index: O(1) lookup by full symbol ID
  nodesById: Map<string, D3Node>;
  
  // Secondary indices for user queries
  nodesByDisplayName: Map<string, D3Node[]>;  // "my_function" → [node1, node2]
  nodesByFile: Map<string, D3Node[]>;         // "src/module.rs" → [...]
  nodesByLine: Map<string, Map<number, D3Node>>; // file → line → node
  
  // Metadata
  metadata: {
    totalNodes: number;
    totalEdges: number;
    projectRoot: string;
    generatedAt: Date;
    indexVersion: string;  // For compatibility checks
  };
}
```

### Memory Footprint Estimation

For a project with 10,000 functions:
- Average node JSON: ~500 bytes
- Total nodes: ~5MB
- Plus JS object overhead: ~10-15MB total

This is acceptable for a VS Code extension (typical extensions use 50-200MB).

---

## Enrichments from Pipeline

The `pipeline` command from scip-callgraph provides two key enrichments beyond the basic call graph:

### Verification Status

Each node can have a `verification_status` field:

| Status | Meaning | Visual Indicator |
|--------|---------|------------------|
| `verified` | Function passed Verus verification | 🟢 Green border/badge |
| `failed` | Function failed Verus verification | 🔴 Red border/badge |
| `unverified` | Function uses `assume` or `admit` | 🟡 Yellow border/badge |
| (none) | Not a Verus function or not verified | Default styling |

**Pipeline command to include verification:**
```bash
cargo run --release -p metrics-cli --bin pipeline -- /path/to/project
# Verification runs automatically unless --skip-verification is passed
```

**Per-function verification is supported** (but not recommended for incremental updates):
```bash
# Verify entire project (~12s for 909 functions)
cargo verus verify

# Verify single module (~8s for 58 functions)
cargo verus verify -- --verify-only-module edwards

# Verify single function (~7s - high fixed overhead!)
cargo verus verify -- --verify-only-module edwards --verify-function decompress
```

**Why we recommend full verification over per-function:**

| Scope | Functions | Time | Time per fn |
|-------|-----------|------|-------------|
| Full project | 909 | 12.09s | ~13ms |
| Single module | 58 | 8.47s | ~146ms |
| Single function | 1 | 6.73s | 6730ms |

The ~6-7s fixed overhead per Verus invocation means per-function verification only saves ~5s vs full verification, while adding significant complexity. **Just run full verification** - it's fast enough.

**Verus caching**: Verus appears to cache verification results. Subsequent runs after `cargo clean` are faster, which benefits the full verification approach.

### Displaying Verification Status

```
┌─────────────────────────────────────────────────────────────────┐
│  [Graph View]                       │  [Node Details Panel]    │
│                                     │                          │
│     ┌──────────┐                    │  lemma_bytes_roundtrip   │
│     │🟢 A      │────┐               │  ───────────────────────  │
│     └──────────┘    │               │                          │
│          │          │               │  Status: ✅ verified     │
│          ▼          ▼               │  Mode: proof             │
│     ┌──────────┐ ┌──────────┐       │                          │
│     │🟢 B      │ │🔴 C      │       │  Dependencies (2):       │
│     └──────────┘ └──────────┘       │  • spec_to_bytes         │
│                                     │  • lemma_seq_len         │
│  Legend:                            │                          │
│  🟢 = verified  🔴 = failed         │  Dependents (1):         │
│  🟡 = unverified                    │  • main_entry            │
│                                     │                          │
└─────────────────────────────────────────────────────────────────┘
```

**TypeScript helper for rendering:**
```typescript
function getNodeColor(node: D3Node): string {
  switch (node.verification_status) {
    case 'verified': return '#22c55e';   // green
    case 'failed': return '#ef4444';     // red
    case 'unverified': return '#eab308'; // yellow
    default: return '#6b7280';           // gray
  }
}
```

**Filtering options:**
- Show only verified/failed/unverified functions
- Filter by mode (exec/proof/spec)
- Filter by file pattern

---

## Query-time Subgraph Extraction

### Algorithm: Bidirectional BFS

```typescript
interface SubgraphQuery {
  rootFunctionId: string;
  depth: number;
  direction: 'dependencies' | 'dependents' | 'both';
  filters?: {
    excludeExternal?: boolean;
    onlyVerified?: boolean;
    filePattern?: string;
  };
}

interface SubgraphResult {
  nodes: D3Node[];
  edges: Array<{ source: string; target: string; type: string }>;
  rootNode: D3Node;
  stats: {
    totalNodes: number;
    maxDepthReached: number;
    queryTimeMs: number;
  };
}

function extractSubgraph(
  index: CallGraphIndex,
  query: SubgraphQuery
): SubgraphResult {
  const startTime = performance.now();
  
  const visited = new Set<string>();
  const resultNodes: D3Node[] = [];
  const resultEdges: Array<{ source: string; target: string; type: string }> = [];
  
  // Queue entries: [nodeId, currentDepth, direction]
  const queue: Array<[string, number, 'forward' | 'backward']> = [];
  
  // Start from root
  queue.push([query.rootFunctionId, 0, 'forward']);
  if (query.direction === 'both' || query.direction === 'dependents') {
    queue.push([query.rootFunctionId, 0, 'backward']);
  }
  
  let maxDepthReached = 0;
  
  while (queue.length > 0) {
    const [nodeId, depth, dir] = queue.shift()!;
    
    // Skip if already visited or beyond depth
    const visitKey = `${nodeId}:${dir}`;
    if (visited.has(visitKey) || depth > query.depth) continue;
    visited.add(visitKey);
    
    const node = index.nodesById.get(nodeId);
    if (!node) continue;
    
    // Apply filters
    if (query.filters?.excludeExternal && node.relative_path.startsWith('external:')) {
      continue;
    }
    
    // Add node to result (avoid duplicates)
    if (!resultNodes.find(n => n.id === nodeId)) {
      resultNodes.push(node);
    }
    
    maxDepthReached = Math.max(maxDepthReached, depth);
    
    // Traverse in the appropriate direction
    if (dir === 'forward' && (query.direction === 'dependencies' || query.direction === 'both')) {
      for (const depId of node.dependencies) {
        resultEdges.push({ source: nodeId, target: depId, type: 'calls' });
        queue.push([depId, depth + 1, 'forward']);
      }
    }
    
    if (dir === 'backward' && (query.direction === 'dependents' || query.direction === 'both')) {
      for (const depId of node.dependents) {
        resultEdges.push({ source: depId, target: nodeId, type: 'calls' });
        queue.push([depId, depth + 1, 'backward']);
      }
    }
  }
  
  // Deduplicate edges
  const edgeSet = new Set(resultEdges.map(e => `${e.source}→${e.target}`));
  const uniqueEdges = resultEdges.filter((e, i) => {
    const key = `${e.source}→${e.target}`;
    return resultEdges.findIndex(e2 => `${e2.source}→${e2.target}` === key) === i;
  });
  
  return {
    nodes: resultNodes,
    edges: uniqueEdges,
    rootNode: index.nodesById.get(query.rootFunctionId)!,
    stats: {
      totalNodes: resultNodes.length,
      maxDepthReached,
      queryTimeMs: performance.now() - startTime,
    },
  };
}
```

### Complexity Analysis

- **Time**: O(d × k^d) where d = depth, k = average branching factor
  - For typical code: k ≈ 3-5, d ≈ 3-5
  - Worst case with d=5, k=5: 5^5 = 3,125 nodes visited
  - In practice: 50-200 nodes for most queries
  
- **Space**: O(n) for the result set, where n = nodes in subgraph

---

## Updates: Just Re-run the Pipeline

After careful consideration, we chose the **simplest approach**: re-run the entire `scip-callgraph pipeline` on save. This is better than incremental LSP-based updates because:

| Aspect | Re-run Pipeline | Incremental LSP Diffs |
|--------|----------------|----------------------|
| **Implementation** | 🟢 Very simple (shell command) | 🔴 Complex (custom diffing logic) |
| **Correctness** | 🟢 Guaranteed (ground truth) | 🟡 Potential drift/bugs |
| **Maintenance** | 🟢 Low (one code path) | 🔴 High (two paths) |
| **Verification** | 🟢 Included automatically | 🟡 Separate step needed |
| **Time** | 🟡 ~30-60s (background) | 🟢 ~100ms graph + ~12s verify |

**Key insight**: The pipeline runs in the background, so the user isn't blocked. The complexity of incremental updates isn't worth it when you need periodic full regeneration anyway to prevent drift.

### Update Strategy

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    SIMPLE APPROACH: Pipeline on Save                     │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  User saves file (Ctrl+S)                                                │
│         │                                                                │
│         ▼                                                                │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │ Debounce (wait 2-5s for more saves)                               │   │
│  └────────┬─────────────────────────────────────────────────────────┘   │
│           │                                                              │
│           ▼                                                              │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │ BACKGROUND JOB: Run pipeline                                      │   │
│  │                                                                   │   │
│  │ scip-callgraph pipeline /path/to/project \                        │   │
│  │   -o .vscode/call_graph_index.json \                              │   │
│  │   --skip-similar-lemmas                                           │   │
│  │                                                                   │   │
│  │ Shows: "🔄 Updating call graph..." in status bar                  │   │
│  └────────┬─────────────────────────────────────────────────────────┘   │
│           │                                                              │
│           ▼                                                              │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │ Reload index into memory                                          │   │
│  │ Show: "✅ Call graph updated" (with timestamp)                   │   │
│  │ Refresh any open graph views                                      │   │
│  └──────────────────────────────────────────────────────────────────┘   │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### UI: Show "Last Updated" Timestamp

The graph view should clearly show when the data was generated:

```
┌─────────────────────────────────────────────────────────────────────────┐
│  Call Graph: my_function                              [↻ Refresh] [⚙]  │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│              ┌──────────┐                                                │
│              │🟢 caller │                                                │
│              └────┬─────┘                                                │
│                   │                                                      │
│                   ▼                                                      │
│          ┌────────────────┐                                              │
│          │ my_function ★  │  ← selected                                  │
│          └───────┬────────┘                                              │
│                  │                                                       │
│         ┌────────┴────────┐                                              │
│         ▼                 ▼                                              │
│    ┌──────────┐      ┌──────────┐                                        │
│    │🟢 helper │      │🔴 util   │                                        │
│    └──────────┘      └──────────┘                                        │
│                                                                          │
├─────────────────────────────────────────────────────────────────────────┤
│  📊 Graph generated: Dec 26, 2025 at 2:45 PM (3 minutes ago)            │
│  ⚠️ Unsaved changes may not be reflected                                │
└─────────────────────────────────────────────────────────────────────────┘
```

When an update is in progress:
```
├─────────────────────────────────────────────────────────────────────────┤
│  🔄 Updating graph... (pipeline running)                                │
│  📊 Current data from: Dec 26, 2025 at 2:42 PM                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### Why Not Incremental LSP Updates?

We initially considered using verus-analyzer LSP for incremental graph updates. However:

1. **Complexity**: LSP-based diffing requires handling many edge cases (renames, macros, etc.)
2. **Correctness**: Incremental updates can drift from ground truth over time
3. **Verification already runs**: We need full verification (~12s) anyway
4. **Pipeline is comprehensive**: Handles SCIP, graph, AND verification in one command

If projects become large enough that pipeline time is problematic (>5 minutes), we could revisit incremental updates. For now, simplicity wins.

---

**Historical context (for reference)**: If we ever implement incremental updates, we'd use verus-analyzer LSP (not tree-sitter) because:

| Aspect | verus-analyzer (LSP) | tree-sitter-verus |
|--------|---------------------|-------------------|
| **Symbol Resolution** | ✅ Full semantic understanding - knows `helper()` → `crate::utils::helper` | ❌ Syntax only - just sees text "helper()" |
| **Already Running** | ✅ Already active in VS Code | ❌ Need to load/configure separately |
| **SCIP Compatibility** | ✅ Same symbols as the index (both from verus-analyzer) | ❌ Need manual symbol mapping |
| **Verus Understanding** | ✅ Knows `spec fn`, `proof fn`, `requires`, `ensures` | ✅ Can parse them (grammar level) |
| **Offline Use** | ❌ Needs language server running | ✅ Works standalone |
| **Speed** | ~10-50ms per LSP request | ~1-5ms parse |

**The critical issue with tree-sitter**: It only gives you syntax, not semantics.

```rust
// In src/module_a.rs
fn process() {
    helper();        // tree-sitter sees: "helper()" - but WHICH helper?
    utils::helper(); // tree-sitter sees: "utils::helper()" - still ambiguous
    self.foo();      // tree-sitter sees: "self.foo()" - what type is self?
}
```

To update the index correctly, we need to resolve these to their full SCIP symbols:
- `helper()` → `rust-analyzer cargo my_crate 0.1.0 utils#helper().`
- `self.foo()` → `rust-analyzer cargo my_crate 0.1.0 module_a/MyStruct#foo().`

**verus-analyzer already knows this** through its semantic analysis. Using tree-sitter would require reimplementing name resolution - complex and error-prone.

> **Note**: [tree-sitter-verus](https://github.com/secure-foundations/tree-sitter-verus) is useful for syntax highlighting, code folding, and offline tools, but not for semantic updates in a VS Code extension where verus-analyzer is already available.

### Update Scenarios

With the simple "re-run pipeline" approach, all scenarios are handled uniformly:

| Scenario | Action |
|----------|--------|
| Any code change | Re-run pipeline on save (debounced) |
| Cargo.lock changed | Re-run pipeline |
| Manual refresh | User command to re-run pipeline |

**The pipeline handles everything**: SCIP generation, call graph building, and verification status.

### Pipeline Command Details

The extension runs scip-callgraph's `pipeline` command:

```bash
# From scip-callgraph directory (or with installed binary)
cargo run --release -p metrics-cli --bin pipeline -- \
  /path/to/verus-project \
  -o .vscode/call_graph_index.json \
  --skip-similar-lemmas
```

**Command-line options:**

| Option | Description |
|--------|-------------|
| `-o, --output <path>` | Output file (default: `web/public/graph.json`) |
| `--skip-verification` | Skip Verus verification (faster, no status) |
| `--skip-similar-lemmas` | Skip similar lemmas (no Python needed) |
| `--use-cached-scip` | Reuse existing `index.scip.json` if available |
| `-p, --package <name>` | For workspaces, specify the crate |
| `--github-url <url>` | GitHub URL for source code links |

**What the pipeline does (automatically):**

1. **Generate SCIP index**: `verus-analyzer scip .` → `index.scip`
2. **Convert to JSON**: `scip print --json` → `index.scip.json`
3. **Build call graph**: Extract atoms, build D3 graph with dependencies/dependents
4. **Run verification**: `cargo verus verify` → enriches with `verification_status`
5. *(Optional)* **Similar lemmas**: Python enrichment

**Prerequisites:**
- `verus-analyzer` in PATH
- `scip` CLI in PATH (`cargo install scip-cli`)
- `cargo verus` for verification

**Time estimate:** ~30-60 seconds for typical projects (includes ~12s for verification)

### Recommended Implementation Phases

| Phase | What | Complexity |
|-------|------|------------|
| **Phase 1** | Manual regeneration via command | Low |
| **Phase 2** | Auto-regenerate on save (debounced, background) | Low |
| **Phase 3** | Show "last updated" timestamp in graph view | Low |

All phases use the simple "re-run pipeline" approach. No incremental LSP updates needed.

---

## Implementation Plan

### Phase 1: Core Index-based Architecture

**Files to create:**

1. `src/indexLoader.ts` - Load and cache the D3 graph index
2. `src/subgraphExtractor.ts` - BFS traversal for subgraph extraction
3. `src/d3Renderer.ts` - Render subgraph as D3 or DOT

**Files to modify:**

1. `src/extension.ts` - Preload index on activation
2. `src/graphGenerator.ts` - Use new index-based approach
3. `package.json` - Update commands, add configuration

**New commands:**

- `callGraph.regenerateIndex` - Regenerate the full index
- `callGraph.showDependencies` - Show what the function calls
- `callGraph.showDependents` - Show what calls the function
- `callGraph.showBidirectional` - Show both directions

### Phase 2: Auto-regenerate on Save

**Implementation:**
1. Add file watcher for `*.rs` files (VS Code `workspace.onDidSaveTextDocument`)
2. Debounce saves (wait 2-5s for batch saves)
3. Run scip-callgraph `pipeline` in background
4. Show status bar indicator: "🔄 Updating call graph..."
5. Reload index into memory when complete
6. Show notification: "✅ Call graph updated"

**Configuration options:**
```json
{
  "callGraph.autoRegenerateOnSave": true,
  "callGraph.debounceDelayMs": 3000,
  "callGraph.skipVerification": false,
  "callGraph.skipSimilarLemmas": true
}
```

### Phase 3: Show "Last Updated" in Graph View

**Implementation:**
1. Store `generatedAt` timestamp in index metadata
2. Display in graph view footer: "📊 Graph generated: Dec 26, 2025 at 2:45 PM"
3. Show warning if unsaved changes: "⚠️ Unsaved changes may not be reflected"
4. Show progress when updating: "🔄 Updating graph... (pipeline running)"

### Directory Structure (Updated)

```
call_graph_vs_code_extension/
├── src/
│   ├── extension.ts           # Entry point, command registration
│   ├── indexLoader.ts         # NEW: Load/cache D3 graph index
│   ├── subgraphExtractor.ts   # NEW: BFS subgraph extraction
│   ├── graphRenderer.ts       # NEW: Render subgraph (replaces graphGenerator.ts)
│   ├── pipelineRunner.ts      # NEW: Run scip-callgraph pipeline
│   ├── webview/
│   │   ├── graphView.html     # Interactive D3.js view
│   │   └── graphView.ts       # Webview script
│   └── test/
│       ├── indexLoader.test.ts
│       └── subgraphExtractor.test.ts
├── package.json
├── DESIGN_PRECOMPUTED_INDEX.md  # This document
└── README.md
```

---

## Open Questions

1. **Binary Distribution**: Should we bundle scip-callgraph binaries with the extension, or require users to install separately?
   - Bundling: Simpler UX, larger extension size (~10MB)
   - Separate: Smaller extension, more setup steps
   - **Recommendation**: Provide both options - auto-download on first use

2. **Index Location**: Where should `call_graph_index.json` be stored?
   - Workspace root: Easy to find, clutters project
   - `.vscode/`: Hidden, standard location for VS Code config
   - Global cache: Shared across workspaces, harder to manage
   - **Recommendation**: `.vscode/call_graph_index.json` with option to configure

3. **Symbol Matching**: How do we match the user's cursor position to a SCIP symbol?
   - Use verus-analyzer LSP's `textDocument/definition` to get the exact symbol at cursor
   - Fall back to matching by file + line range from the index
   - **Note**: verus-analyzer provides accurate symbol resolution - this is why we use it over tree-sitter

4. **Large Projects**: For projects with 100k+ functions, should we:
   - Lazy-load parts of the index?
   - Use a more compact binary format (e.g., MessagePack)?
   - Use SQLite instead of JSON?
   - **Recommendation**: Start with JSON, profile real-world usage, optimize if needed

5. **Verus Support**: Special handling for Verus annotations:
   - ✅ Different colors for `spec`, `proof`, `exec` functions (already in D3Graph format via `mode` field)
   - ✅ Verification status via `cargo verus verify` (full project, ~12s)
   - Show `requires`/`ensures` in tooltips (scip-callgraph uses `verus_syn` to parse these)
   - **Note**: verus-analyzer does NOT provide verification status - we must run `cargo verus verify`
   - **Note**: Verus supports per-function verification (`--verify-function <f>`) but has ~6s fixed overhead, so full verification is simpler

6. **Update Strategy**: When to regenerate the index?
   - **On save (debounced)**: Re-run pipeline after 2-5s of inactivity
   - **On demand**: User command to force regeneration
   - **On project open**: Check staleness, prompt if outdated
   - **Decision**: We chose full pipeline regeneration over incremental LSP updates for simplicity. The pipeline runs in ~30-60s (background), which is acceptable.

7. **Verus Caching**: Verus appears to cache verification results internally
   - Subsequent runs after `cargo clean` are faster
   - This makes full verification even more practical
   - **TODO**: Investigate Verus caching behavior in more detail

---

## References

### Core Tools
- [scip-callgraph](https://github.com/Beneficial-AI-Foundation/scip-callgraph) - Call graph generation from SCIP indices
  - `pipeline` binary: Unified command for generating enriched call graphs
- [verus-analyzer](https://github.com/verus-lang/verus-analyzer) - Fork of rust-analyzer with Verus support
- [SCIP Protocol](https://github.com/sourcegraph/scip) - Source Code Intelligence Protocol

### VS Code Extension Development
- [VS Code Extension API](https://code.visualstudio.com/api) - Official extension development guide
- [Language Server Protocol](https://microsoft.github.io/language-server-protocol/) - LSP specification

### Related Tools (for reference)
- [verus_syn](https://github.com/verus-lang/verus/tree/main/source/verus_syn) - Verus syntax parser (used by scip-callgraph to extract `requires`/`ensures`)
- [tree-sitter-verus](https://github.com/secure-foundations/tree-sitter-verus) - Verus grammar for tree-sitter (useful for syntax highlighting, not for semantic updates)
- [rust-analyzer](https://rust-analyzer.github.io/) - Rust language server (verus-analyzer is forked from this)

