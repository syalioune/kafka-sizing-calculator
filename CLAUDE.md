# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

A single-file, zero-dependency Kafka cluster sizing calculator. The entire application lives in `index.html` — one self-contained HTML file with inline CSS and JavaScript. No build tools, no package manager, no framework. Open the file in a browser to run it. Deployed to GitHub Pages via `.github/workflows/deploy.yml`.

## Architecture

The app has two top-level modes, toggled via mode tabs:

1. **Sizing Calculator** — a tabbed calculator with 10 tabs: Inputs, Throughput, Storage, Brokers, Cluster Topology, Partitions, Schema Registry, REST Proxy, Scenarios, Monitoring, and Summary.
2. **Architecture Diagram Builder** — a manual diagram builder for designing Kafka architecture diagrams with custom DC layout, components, storage visualization, and PNG/SVG export.

**Key structure within the single file:**
- **Lines 1–222**: CSS (custom properties in `:root`, responsive grid, card/form/table/mode-tab component styles)
- **Lines 224–505**: HTML — mode tabs, calculator tabs and content sections, diagram builder inputs and output
- **Lines 507–end**: JavaScript — all calculation logic, diagram rendering, and DOM management

**Core JS functions:**
- `showMode(mode)` — toggles between 'calculator' and 'diagram' top-level modes
- `recalc()` — the main function, triggered by every input change. Reads all inputs via `v(id)`, computes throughput/storage/broker/topology/partition metrics, and renders results into DOM containers
- `renderArchDiagram()` — reads diagram builder inputs and renders architecture diagram using the same HTML structure as topology diagrams (dc-box, dc-comp, arrow-col, cluster-wrap/cluster-indep, mesh-link)
- `exportDiagram(diagId, fmt)` — exports any diagram container (stretch-diagram, multi-diagram, or diag-output) as SVG or PNG using native SVG elements and Canvas 2D API. Auto-detects stretch vs multi topology from DOM content
- `scenarioCalc(s)` — independent calculation engine for the Scenarios tab (takes a scenario object, returns computed metrics)
- `renderScenarios()` / `renderMon()` — render the scenario comparison table and monitoring thresholds table
- `setTopology(t)` — toggles between "stretch" and "multi" cluster topology views
- `showTab(name, btn)` — tab switching, also triggers `recalc()`
- Helper formatters: `fmtMBs()`, `fmtTB()`, `fmtNum()`, `fmtEvt()`, `rr()` (result row), `mc()` (metric card), `banner()` (input summary strip), `storageBar()` (hot/cold storage visualization)

**Calculation flow (Sizing Calculator):** Input values → throughput (ingress, replication, consumer egress, total I/O) → storage (raw, compressed, with RF, hot/cold tiers) → broker count (max of network-bound, disk-bound, storage-bound, min RF) → topology (stretch vs multi-cluster sizing with DC distribution, storage visualization) → partitions → Schema Registry sizing → REST Proxy sizing → summary aggregation (including SR/RP in per-DC totals).

**Architecture Diagram Builder:** User specifies topology (stretch/multi), DC count, custom DC names, per-link inter-DC labels, components per DC (brokers, controllers, SR, RP, MM2), and storage visualization mode. The diagram renders live using the same CSS classes and HTML structure as the topology tab. Supports PNG/SVG export via the shared `exportDiagram()` function.

## Conventions

- All DOM rendering uses template literal strings with `innerHTML` assignment — no virtual DOM or templating library
- CSS uses custom properties (`var(--accent-blue)`, etc.) for theming consistency
- Tooltip content is stored in `data-tip` attributes on `.tip` spans
- Input fields use `oninput="recalc()"` for live recalculation
- The scenario comparison table has its own independent state (`scenarioState` array) separate from the main inputs

## License

Apache 2.0
