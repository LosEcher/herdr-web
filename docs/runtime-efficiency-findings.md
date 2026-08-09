# Runtime Efficiency Findings

Analysis of **data storage**, **data structures**, **algorithms**, **messaging**, and **performance**
on the current trunk. Ordered by “clear upside vs effort”, not by theoretical purity.

Legend:

- **Impact:** user-visible or operator-visible gain if fixed
- **Effort:** rough engineering cost
- **When:** scale or scenario where it starts to matter

---

## 1. Data Storage

### Current model

| Store | Location | Format | Access pattern |
|---|---|---|---|
| Notes | `XDG_DATA_HOME/herdr-web/notes` (etc.) | JSON array + lock + atomic rename | Full read → mutate → full rewrite |
| Observations (note linking) | sibling JSON | JSON array, pruned to 5k/session | Full rewrite on change |
| Agent pins | JSON + lock | Vec/list filtered by `session_key` | Full rewrite |
| Web Push subs + VAPID | `herdr-web-push/` | JSON; subs in-memory `HashMap` | Full rewrite on subscribe/prune |
| Launcher presets | config path | JSON file load | Read-mostly |
| Uploads | upload dir | raw files | size-capped |
| Client prefs / bridge profiles | `localStorage` (+ native prefs path) | JSON blobs | Read on boot, write on change |
| Display/nav/notification prefs | mostly browser-local | fragmented keys | No bridge sync |

Strengths: private file modes (unix 0600/0700), atomic writes, corrupt-copy once, session isolation
via `session_key()`, revision checks on notes.

### Clear storage improvements

| Opportunity | Impact | Effort | When | Notes |
|---|---|---|---|---|
| **Notes: index by `note_id` + avoid full-file rewrite every mutation** | Med | Med | hundreds of notes | Today `NotesStore.notes: Vec` + rewrite whole file on each update/attach. Fine for tens of notes; wasteful for large bodies (max 256 KiB body × N). Consider `HashMap` in memory + append-only or per-note files, or SQLite. |
| **Notes list API: server-side filter/pagination** | Med | Low–Med | large note lists | Client often gets full list then filters. Bridge already has query fields; ensure they short-circuit and avoid serializing deleted/archived when not requested. |
| **Observation store growth** | Low–Med | Low | long-lived sessions | Cap is 5k observations/session; prune exists. Worth metrics: rewrite size, prune frequency. |
| **Push subscriptions: durable index + no silent oldest-drop only** | Low | Low | multi-device | `MAX_SUBSCRIPTIONS=32` drops oldest by map order. Prefer LRU by `last_seen` + UI to revoke. |
| **Unify client prefs → optional bridge-backed prefs blob** | Med (UX) | Med | multi-device | Storage is fragmented across localStorage keys. One versioned prefs document on bridge would fix multi-browser drift (alerts, display). Not a perf win first—consistency win. |
| **Do not introduce SQLite until notes/prefs actually hurt** | — | — | — | JSON stores are appropriate for local-first small data. Premature DB adds packaging/sync complexity. |

**Verdict:** storage is **correct for the product scale**. Biggest *clear* wins are (1) notes mutation I/O shape if notes grow, (2) prefs consolidation for multi-device, not a rewrite of the store layer.

---

## 2. Data Structures

### Snapshot (browser truth)

```text
Snapshot {
  workspaces: WorkspaceInfo[]
  tabs: TabInfo[]
  panes: PaneInfo[]
  layouts: LayoutSnapshot[]
  selected_pane_id, ...
}
```

- Arrays are the wire and in-memory shape.
- Lookups are overwhelmingly `find` / `findIndex` / `filter` by id (**O(n)** per query).
- Activity patch rebuilds `panes`, then remaps **all** workspaces and tabs to recompute
  aggregate status (see `applyPaneAgentStatusChanged` in `web/src/activity.ts`).

### Bridge

- Terminal sessions: `HashMap<pane_id, SharedTerminalSession>` — good.
- Activity tracker: `HashMap<PaneKey, …>` + `pane_index: HashMap<pane_id, …>` — good.
- Notes / observations: `Vec` — simple, linear scans.
- Push: `HashMap<endpoint, subscription>` — good.
- Broadcast channels: activity capacity **512**, ui-events **256**.

### Frontend UI state

- `Record<bridgeId, …>` for connection / activity / pins / notes — right shape for multi-host.
- Sidebar grouping rebuilds visible lists via nested map/filter/sort in `App` (`buildVisible*`).
- No persistent id→entity indexes beside ad-hoc `useMemo`.

### Clear structure improvements

| Opportunity | Impact | Effort | When | Notes |
|---|---|---|---|---|
| **Maintain `Map<pane_id, index\|pane>` beside snapshot arrays** | Med | Low | 50+ panes | Hot paths: selection, activity patch, agent list, pin lookup. Keep arrays for render order; derive maps in one place when snapshot changes. |
| **Activity patch: only touch affected workspace/tab** | Med | Low | frequent status flips | Today maps every workspace/tab and re-filters panes per aggregate. Patch one pane + recompute one workspace + one tab (O(panes_in_tab) not O(all)). |
| **Notes in-memory index by id** | Low–Med | Low | many notes | Avoid linear `position` scans on every mutation. |
| **Agent sidebar: pre-bucket by workspace/status** | Med | Med | large herds | `buildVisibleAgentPaneEntries` sorts/groups repeatedly; cache keyed by snapshot generation + sort prefs. |
| **Avoid cloning full snapshot on every activity event feeding React** | High* | Med | busy agents | *If profiling shows App re-render cost. Structural sharing helps only if consumers don't deep-depend on new array identities everywhere—pair with selective context. |

**Verdict:** wire format can stay arrays (JSON-friendly). **Derived indexes** are the low-hanging fruit; changing the protocol shape is not required.

---

## 3. Algorithms

### Already solid

- **Directional pane focus** (`chooseDirectionalPane`) — score + sort candidates; fine for layout size.
- **Snapshot refresh single-flight** (`refreshCoordinator`) — coalesces concurrent refresh triggers; generation + barrier for stale apply.
- **Activity replay log** — generation-stamped; replays only newer deltas after snapshot fetch.
- **Terminal output coalescing** (bridge + optional client) — time window (default 16ms), max bytes 32KiB, max chunks 256; metrics hooks exist.
- **Activity resubscribe** — compare sorted pane id sets; skip rebuild when unchanged.
- **Workspace reorder / grouping** — pure helpers with tests.

### Clear algorithmic improvements

| Opportunity | Impact | Effort | When | Notes |
|---|---|---|---|---|
| **`current_panes` / snapshot path: cache + invalidate** | High | Med | many HTTP actions | Notes/pins handlers repeatedly call `current_panes(&api)` (blocking Herdr IPC). Cache short-lived pane list invalidated by structural events / TTL. |
| **Snapshot handler payload size** | Med | Med | large sessions | Full snapshot every recovery/refresh. Optional compact snapshot (no layouts when sidebar-only) or ETag/generation would cut JSON parse + React apply cost. |
| **Aggregate status: incremental** | Low–Med | Low | activity storms | `aggregateStatus` over filtered panes is cheap until pane counts are large; combine with targeted patch. |
| **Agent sort `lastStatusChange`** | Low | Low | attention sort | Ensure transitions map is O(1) lookup per pane (already memoized transitions in App—keep that path, avoid resorted scans). |
| **Push broadcast: sequential HTTP** | Med | Low–Med | many subs | `broadcast_alert` should fan-out concurrently (bounded join set), not one-by-one await if it currently serializes. |
| **Don't over-optimize pure O(n) list UI** | — | — | &lt;100 panes | Premature index everywhere increases complexity; profile first with 100/500 pane fixtures. |

**Verdict:** largest algorithmic win is **reduce Herdr `pane.list` / full snapshot churn on bridge**, not micro-optimizing sort comparators.

---

## 4. Messaging / Communication

### Channels (bridge ↔ browser)

| Channel | Role | Lag policy |
|---|---|---|
| `/ws/events` | structural / selection-ish Herdr-driven events | (existing handler path) |
| `/ws/activity` | agent status deltas | **Lagged → `resync_required` + close** (correct, costly) |
| `/ws/ui-events` | notes/pins/activity changed pings | Lagged → **continue** (drop) |
| `/ws/terminal` | binary/json terminal I/O | coalesced broadcast of output |
| HTTP `/api/*` | snapshot, command, notes, pins, push, uploads | request-gated |

Internal:

- `activity_tx` broadcast **512**
- `ui_event_tx` broadcast **256**
- Terminal session `output_tx` broadcast per shared session
- Client input via `mpsc` with large byte caps

### Clear messaging improvements

| Opportunity | Impact | Effort | When | Notes |
|---|---|---|---|---|
| **Activity lag: coalesce latest-per-pane instead of hard resync** | **High** | Med | tab backgrounded / mobile | Closing the socket forces full snapshot refresh. A ring buffer or `HashMap<pane_id, latest_status>` latest-wins coalescer would drop intermediates but avoid resync storms. |
| **ui-events: typed binary or shared enum, not ad-hoc JSON strings** | Low–Med | Med | debugability | Today string payloads; fine at low rate. Typed messages reduce parse mistakes. |
| **Differentiate “ping to refetch” vs “payload embedded”** | Med | Med | notes multi-client | Notes changes broadcast then clients refetch lists. Embedding note_id/revision (partially present) + optional small payload cuts RTT. |
| **Terminal multi-viewer: shared session already** | — | — | — | Good design; watch detach/drain paths under many viewers. |
| **Multi-bridge client: N sockets × 4** | Med | Med | many hosts | Each enabled runtime opens events/activity/ui/terminal as needed. Connection budget and battery on mobile; consider multiplex later only if N hosts &gt; 3 common. |
| **Web Push vs live activity** | Low | Low | — | Push is out-of-band; ensure live path doesn't also spam Notification when tab focused (prefs already gate—keep single policy module). |
| **Backpressure metrics export** | Med (ops) | Low | production | Coalesce/lag counters exist in bridge; expose `/api/metrics` or log periodic summaries so lag is visible without code diving. |

**Verdict:** messaging **architecture is sound**. The one **obvious** product-perf footgun is **activity lag → full resync**. Prefer latest-status coalescing before growing channel capacities blindly.

---

## 5. Performance (render / CPU / I/O)

### Hot paths

1. **Terminal output** — Herdr → bridge coalesce → WS → Ghostty write (already tuned; 16ms default ≈ 1 frame).
2. **Agent status storms** — activity WS → snapshot patch → App state → sidebar recompute → possible notifications/push.
3. **Full snapshot refresh** — JSON parse + replace connection state + rebuild all memos in `App`.
4. **App render fan-out** — huge component with ~90 `useState` / ~57 `useEffect`; any connectionState update can re-render large trees.
5. **Bridge blocking IPC** — `spawn_blocking` + `current_panes` on many API routes.

### Clear performance improvements

| Opportunity | Impact | Effort | When | Notes |
|---|---|---|---|---|
| **Split React state contexts** (connection vs selection vs notes vs settings) | **High** | Med–High | always | Biggest UI jank lever. Terminal should not re-render on notes list updates. |
| **Memoized row components with stable callbacks** | Med | Low–Med | long sidebars | `AgentRow`/`PaneRow` exist; ensure parent doesn't pass fresh lambdas every render. |
| **Virtualize Agents/Tabs lists** | Med | Med | 200+ rows | Only after state split; virtualization on a thrashing parent is wasted. |
| **Activity → setState batching** | Med | Low | status storms | Ensure multiple rapid deltas don't each force layout; rAF-batch pane patches optional. |
| **Double coalesce (bridge + client)** | Low | Low | misconfig | Defaults 16ms both sides can add latency; document “prefer bridge-side only” for LAN. |
| **Snapshot apply: structural share + generation gate** | Med | Med | refresh-heavy | Already has generation/barrier; keep single-flight; avoid cloning layouts when only agent fields changed (ties to activity patch). |
| **Bridge pane list cache** | High | Med | API chatter | Same as algorithms section. |
| **Push fan-out concurrency** | Med | Low | many devices | Don't block activity watcher on slow push endpoints; already `maybe_broadcast_web_push` spawns task—confirm and bound concurrency. |
| **Profile before micro-opts** | — | Low | — | Add a debug fixture: 100 workspaces × 5 panes, scripted status flips, measure interaction + WS lag. |

### What not to chase early

- Replacing Ghostty or rewriting terminal protocol.
- Moving snapshot to binary protobuf (JSON is not the first bottleneck vs React + full resync).
- Global Redux/Zustand rewrite without boundary plan (context split is enough first step).

---

## 6. Priority Stack (recommended)

If the goal is **noticeable improvement with bounded risk**:

1. **Activity lag policy: latest-per-pane coalesce** (messaging) — fewer full snapshot storms. **Done** (side-cache + replay on lag).
2. **React state split / reduce App re-render scope** (performance) — jank and battery. *(next)*
3. **Bridge `current_panes` / snapshot short-lived cache** (algorithms + I/O) — less Herdr IPC. **Done** (300ms TTL for store handlers).
4. **Activity patch only affected workspace/tab + pane id map** (structures + algorithms). **Done** (targeted patch; full pane index still optional).
5. **Notes prefs/store only if user note volume or multi-device prefs hurt** (storage).
6. **List virtualization** after 1–4.

Structural refactors (`App` / `web_bridge` splits) **enable** 2–4 but can ship incrementally with them.

---

## 7. Measurement Checklist

Before claiming a win:

- [ ] Pane count fixture (10 / 100 / 500)
- [ ] Activity flip rate (1/s, 20/s, 100/s across panes)
- [ ] Bridge logs: coalesce flush stats, lagged_events, activity resync count
- [ ] Browser: React profiler commit times on status flip vs keypress in terminal
- [ ] Notes: file size + mutation latency at 10 / 100 / 1000 notes
- [ ] Multi-bridge: 1 vs 3 hosts socket count and CPU

---

## 8. Relation To Architecture Findings

See `docs/architecture-findings.md`. Efficiency work should not expand `App.tsx` / `web_bridge.rs`
further without extracting a home for the change. Prefer:

- `web/src/activity.ts` + small index helpers for patch/index work
- new `bridge/src/pane_cache.rs` or section in a split watcher module for pane list cache
- context modules under `web/src/` for state split
