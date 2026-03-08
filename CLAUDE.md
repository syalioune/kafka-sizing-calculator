# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

A single-file, zero-dependency Kafka cluster sizing calculator. The entire application lives in `index.html` — one self-contained HTML file with inline CSS and JavaScript. No build tools, no package manager, no framework. Open the file in a browser to run it. Deployed to GitHub Pages via `.github/workflows/deploy.yml`.

## Architecture

The app is a tabbed calculator with 10 tabs: Inputs, Throughput, Storage, Brokers, Cluster Topology, Partitions, Schema Registry, REST Proxy, Scenarios, Monitoring, and Summary.

**Key structure within the single file:**
- **Lines 1–188**: CSS (custom properties in `:root`, responsive grid, card/form/table component styles)
- **Lines 190–364**: HTML — tab navigation and content sections, each tab is a `<div id="tab-{name}">` toggled via `.active` class
- **Lines 365–795**: JavaScript — all calculation logic and DOM rendering

**Core JS functions:**
- `recalc()` — the main function, triggered by every input change. Reads all inputs via `v(id)`, computes throughput/storage/broker/topology/partition metrics, and renders results into DOM containers
- `scenarioCalc(s)` — independent calculation engine for the Scenarios tab (takes a scenario object, returns computed metrics)
- `renderScenarios()` / `renderMon()` — render the scenario comparison table and monitoring thresholds table
- `setTopology(t)` — toggles between "stretch" and "multi" cluster topology views
- `showTab(name, btn)` — tab switching, also triggers `recalc()`
- Helper formatters: `fmtMBs()`, `fmtTB()`, `fmtNum()`, `fmtEvt()`, `rr()` (result row), `mc()` (metric card), `banner()` (input summary strip), `storageBar()` (hot/cold storage visualization)

**Calculation flow:** Input values → throughput (ingress, replication, consumer egress, total I/O) → storage (raw, compressed, with RF, hot/cold tiers) → broker count (max of network-bound, disk-bound, storage-bound, min RF) → topology (stretch vs multi-cluster sizing with DC distribution, storage visualization) → partitions → Schema Registry sizing → REST Proxy sizing → summary aggregation (including SR/RP in per-DC totals).

## Conventions

- All DOM rendering uses template literal strings with `innerHTML` assignment — no virtual DOM or templating library
- CSS uses custom properties (`var(--accent-blue)`, etc.) for theming consistency
- Tooltip content is stored in `data-tip` attributes on `.tip` spans
- Input fields use `oninput="recalc()"` for live recalculation
- The scenario comparison table has its own independent state (`scenarioState` array) separate from the main inputs

## License

Apache 2.0
