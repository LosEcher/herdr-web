# Architecture Findings (los/main)

Recorded after a codebase-memory (CBM) index of `herdr-web` on the personal trunk.
Index project: `Users-echerlos-syncthing-project-herdr-web` (~2246 nodes / ~7132 edges).
This note is analysis, not a commitment to implement every item.

## Product Shape

`herdr-web` is a **local-first browser shell** for a running Herdr daemon:

- `web/` — React + Vite multi-host UI
- `bridge/` — repo-owned HTTP/WebSocket adapter (`herdr-web-bridge`)
- `vendor/herdr-compat/` — minimal private Herdr protocol/API surface (protocol `19`)
- `android/` — Capacitor shell around bundled `web/dist`

Browsers never talk to Herdr private APIs directly. The bridge owns attach, snapshot, command
allow-listing, activity subscriptions, and local extension stores (notes, pins, push).

## Design Pillars (what works)

1. **Bridge-owned Herdr coupling** — keep protocol churn out of the browser.
2. **Snapshot for structure, delta for activity** — see `docs/agent-activity-efficiency-design.md`.
3. **One activity watcher per bridge process** — client count does not multiply Herdr subscriptions.
4. **Narrow browser command surface** — validate/allow-list in `web_bridge.rs`.
5. **Pure helpers + unit tests** on the web edge (`state`, `activity`, `commands`, notifications, …).
6. **Local extension stores** with private dirs, atomic JSON write, session keys (`store_util`).
7. **Trusted-network security model** — no browser auth; origin/host allow-lists; documented risk.

## Structural Hotspots (CBM + line counts)

| Symbol / file | Scale | Risk |
|---|---|---|
| `web/src/App.tsx` | ~9.8k lines; `App` ~cyclomatic 309 / cognitive 469 / out-degree 121 | God component; every feature lands here |
| `bridge/src/web_bridge.rs` | ~6.6k lines (~60% of bridge) | Routes, terminal sessions, watchers, request gate |
| `TerminalView.tsx` + `terminalRenderer.ts` | ~3.6k lines combined | Experience-critical, weakly unit-tested |
| `BackendSettingsDialog.tsx` | ~1.5k lines, long prop list | Settings domain not modeled |

CBM clusters (de-facto seams): bridge core, App shell, terminal, activity/request-gate, settings,
bridge provider, notes store. Cohesion is decent *around* the edges; the two centers are overloaded.

## Major Gaps

### Architecture debt (P0)

- Split `App` into resource/nav/notes/sidebar/settings controllers.
- Split `web_bridge` into routes / terminal session / activity watchers / request gate.
- No ADR set before this note; decisions live in README/CHANGELOG/comments.

### Product capability (P1)

- Push: desktop Notification + Web Push exist; Android native alerts and multi-bridge subscription
  consistency are incomplete.
- Prefs mostly client-local; no cross-device sync.
- Secure context required for push → LAN HTTPS story is thin.
- No auth/token path for non-trusted networks (by design today).

### Quality (P2)

- Terminal/renderer and settings UI lack paired unit tests.
- HTTP contract edges are sparse in the graph (dynamic fetch URLs).
- Protocol pin to `19` is correct but makes vendor refresh a recurring cost.

## Branch Strategy (operator)

- **Trunk:** `los/main` on the personal fork (full product line).
- **Upstream:** only stable, reviewable feature slices as separate PRs.
- Do not merge unrelated features into upstream candidate branches.

## Related Upstream PRs (at recording time)

- #51 IME + HMR
- #52 browser desktop notifications
- #53 Web Push (stacked on #52)

## Follow-ups In This Doc Set

- Storage / data structures / algorithms / messaging / performance opportunities:
  `docs/runtime-efficiency-findings.md`

## Efficiency work landed (same session)

- Activity lag: latest-per-pane side-cache + replay (no forced resync).
- Bridge pane-list short cache (300ms) for notes/pins/activity list handlers.
- Frontend activity patch rewrites only affected pane/workspace/tab.
