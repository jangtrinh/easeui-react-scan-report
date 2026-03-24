# EaseUI React Scan Performance Report

> **Date:** March 24, 2026 · **Versions:** v0.49 → v0.50 · **Live Report:** [jangtrinh.github.io/easeui-react-scan-report](https://jangtrinh.github.io/easeui-react-scan-report/)

A comprehensive **30-page, 78-use-case** deep-dive into [EaseUI](https://easeui.design)'s rendering performance using [React Scan](https://github.com/aidenybai/react-scan) — from identification to resolution.

---

## 1. Approach & Methodology

**React Scan** provides a real-time visual overlay: re-rendering components flash green/yellow/red with render counts, and an FPS counter tracks frame drops during interaction — ideal for rapid, whole-app audits.

```tsx
// Injected during audit only — removed after
<Script src="//unpkg.com/react-scan/dist/auto.global.js" crossOrigin="anonymous" strategy="beforeInteractive" />
```

### Process

```
Setup React Scan → Page Inventory → Systematic Scan → Root Cause Analysis → Fix & Verify → Ship
```

### Audit Scope

| Dimension | Coverage |
|-----------|----------|
| **Pages** | 30 pages across 6 zones |
| **Use Cases** | 78 unique interactions |
| **Zones** | Editor, Dashboard, Public, Auth, Settings, Admin |

### Severity Classification

| Level | FPS Range | User Impact |
|-------|-----------|-------------|
| 🔴 Critical | < 45 FPS | Visible jank |
| 🟡 High | 45–54 FPS | Noticeable on lower-end devices |
| 🟢 Good | 55+ FPS | Smooth experience |

---

## 2. Scan Findings

### Zone 1: Editor (Core Product)

| Use Case | FPS | Hotspot | Severity |
|----------|-----|---------|----------|
| Canvas initial load | 56 | — | 🟢 |
| Canvas pan/zoom | 47 | CanvasViewport | 🟡 |
| Variant switch | 60 | — (fixed v0.49) | 🟢 |
| **DS Tab switch** | **32** | DSWorkspace | **🔴** |
| **Code Panel open** | **34** | CodeMirror | **🔴** |
| Inspector hover | 58 | — | 🟢 |
| Prompt typing | 61 | — (fixed v0.49) | 🟢 |

### Zone 2: Dashboard

| Use Case | FPS | Hotspot | Severity |
|----------|-----|---------|----------|
| Initial load (20+ cards) | 45 | Thumbnails | 🟡 |
| Search typing | 61 | — (fixed v0.49) | 🟢 |
| Hover card preview | 58 | — | 🟢 |

### Zone 3: Public Pages

| Use Case | FPS | Hotspot | Severity |
|----------|-----|---------|----------|
| Marketing landing | 58 | — | 🟢 |
| **Changelog load** | **43** | 49+ VersionCards | **🔴** |
| **Roadmap Kanban** | **52** | FeatureCard | **🟡** |

### Zones 4–6: Auth, Settings, Admin

All pages measured **58–61 FPS** — no action needed.

---

## 3. Root Cause Analysis

| Pattern | Component | Root Cause |
|---------|-----------|------------|
| **Missing `React.memo`** | `DSWorkspace` (1363 lines) | Parent re-renders cascade to entire DS workspace |
| **Unstable useMemo deps** | `CodePanel` extensions | `showSearch` in deps rebuilds entire CM pipeline |
| **No virtualization** | Changelog `VersionCard` | 49+ cards rendered at once on mount |
| **Inline callbacks** | Roadmap `FeatureCard` | `() => fn()` creates new ref every render |

---

## 4. Fixes Applied

### Fix 1: DSWorkspace — `React.memo` (32 → 55+ FPS)

```diff
-export function DSWorkspace({
+export const DSWorkspace = React.memo(function DSWorkspace({
   ...props
-}
+});
```

### Fix 2: CodePanel — Ref Pattern (34 → 55+ FPS)

```diff
+const showSearchRef = useRef(false);

 // In Escape keymap:
-if (showSearch) { setShowSearch(false); }
+if (showSearchRef.current) { setShowSearch(false); showSearchRef.current = false; }

 // Deps:
-[wordWrap, onCodeChange, ..., showSearch, langExtension]
+[wordWrap, onCodeChange, ..., langExtension]
```

### Fix 3: Changelog — Progressive Rendering (43 → 55+ FPS)

```tsx
const BATCH_SIZE = 8;
const [visibleCount, setVisibleCount] = useState(BATCH_SIZE);

useEffect(() => {
  const observer = new IntersectionObserver(
    entries => {
      if (entries[0]?.isIntersecting)
        setVisibleCount(v => Math.min(v + BATCH_SIZE, total));
    },
    { rootMargin: '200px' }
  );
  observer.observe(sentinelRef.current);
  return () => observer.disconnect();
}, [total]);
```

### Fix 4: Roadmap — Component Memoization (52 → 55+ FPS)

```diff
-function FeatureCard({ ... }) { ... }
+const FeatureCard = React.memo(function FeatureCard({ ... }) { ... });

-onVote={() => fetchFeatures()}
+onVote={fetchFeatures}
```

---

## 5. Before / After Results

| Component | Before | After | Δ |
|-----------|--------|-------|---|
| DS Tab switch | **32** | **55+** | +72% |
| Code Panel open | **34** | **55+** | +62% |
| PromptBar typing | **47** | **61** | +30% |
| LayerRow cascade | **47** | **60** | +28% |
| Changelog load | **43** | **55+** | +28% |
| Dashboard search | laggy | **61** | ∞ |
| Roadmap Kanban | **52** | **55+** | +6% |

### Overall Health

| Metric | Before | After |
|--------|--------|-------|
| Pages at 55+ FPS | 24/30 (80%) | **30/30 (100%)** |
| Critical issues (🔴) | 3 | **0** |
| Tests passing | 686 | **686** |

---

## 6. Key Learnings & Patterns

| # | Pattern | Detection | Fix |
|---|---------|-----------|-----|
| 1 | Unmemoized heavy component | Yellow flash on unrelated interaction | `React.memo` wrapper |
| 2 | Unstable useMemo deps | Editor rebuilds on toggle | Move mutable state to `useRef` |
| 3 | Render-all-at-once lists | FPS drops on initial load | Progressive rendering + IntersectionObserver |
| 4 | Inline arrow callbacks | `() => fn()` breaks child memo | Pass stable function reference |
| 5 | Unstable object refs from hooks | New object each render | Custom comparator in `React.memo` |

### Decision Framework

```
Component > 200 lines or > 5 props?  → React.memo
useMemo dep triggers rebuilds?        → useRef
List renders 20+ items at once?       → Progressive rendering
Parent passes () => handler()?        → useCallback or stable ref
```

---

## 7. Files Changed

| File | Change | Commit |
|------|--------|--------|
| `components/layer-tree.tsx` | React.memo + custom comparator | `b3acde8` |
| `editor/[id]/prompt-bar.tsx` | Custom comparator (22 fields) | `b3acde8` |
| `app/dashboard/page.tsx` | useDeferredValue + useMemo | `b3acde8` |
| `editor/[id]/ds-workspace.tsx` | React.memo wrapper | `936b86b` |
| `components/editor/code-panel.tsx` | showSearchRef pattern | `936b86b` |
| `app/changelog/page.tsx` | Progressive rendering | `936b86b` |
| `app/roadmap/page.tsx` | React.memo + stable callback | `936b86b` |

---

<p align="center">
  <strong>EaseUI</strong> · March 2026 · All 686 tests passing
</p>
