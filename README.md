# Kafka Cluster Sizing Calculator

A comprehensive, browser-based calculator for sizing Apache Kafka clusters. Derives broker count, hardware specs, storage requirements, partition guidelines, and cross-datacenter infrastructure from your workload parameters.

**[Live Calculator](https://syalioune.github.io/kafka-sizing-calculator/)**

![Apache Kafka](https://img.shields.io/badge/Apache%20Kafka-4.x+-231F20?logo=apachekafka&logoColor=white)
![License](https://img.shields.io/badge/License-Apache%202.0-blue.svg)
![GitHub Pages](https://img.shields.io/badge/Hosted%20on-GitHub%20Pages-222?logo=github)

## Features

- **Zero dependencies** — single self-contained HTML file, no build step, no server required
- **KRaft-native** — all sizing assumes KRaft mode (no ZooKeeper)
- **Tiered Storage** aware — models hot/cold tier split with KIP-405, with object storage requirements in all summaries
- **Multi-DC topology** — compares stretch clusters vs. multiple independent clusters with MirrorMaker 2, with custom DC names and full mesh connectivity diagrams
- **Schema Registry & REST Proxy sizing** — optional component sizing with full integration into infrastructure totals
- **Side-by-side scenario comparison** — 4 fully independent editable workload profiles
- **Architecture Diagram Builder** — design custom Kafka architecture diagrams with manual DC layout, component counts, per-link inter-DC labels, hardware specs per broker, and storage visualization
- **Topology export** — export architecture diagrams as PNG or SVG images
- **Configuration save/load** — name and save any sizing configuration as a shareable URL (unsigned JWT); loading a URL restores all inputs and recomputes results
- **Sizing report** — collapsible copy/paste-ready text report with all formulas, values, and explanations for inclusion in documents
- **Live recalculation** — every input change instantly updates all derived metrics

## Modes

The application has two top-level modes:

- **Sizing Calculator** — derive infrastructure requirements from workload parameters (all sections below)
- **Architecture Diagram Builder** — manually design and export Kafka architecture diagrams (see [Architecture Diagram Builder](#architecture-diagram-builder) section)

## Calculator Sections

### Inputs

The starting point for all calculations. Two input cards:

**Workload Parameters:**
- **Event Rate** — sustained events/sec at steady state
- **Avg Event Size** — serialized message size (key + value + headers) in bytes
- **Peak Multiplier** — burst-to-steady ratio (e.g. 3x means peak is 3 times the steady rate)
- **Compression** — producer codec selection (None, Snappy ~3:1, LZ4 ~4:1, ZSTD ~5:1)

**Replication & Retention:**
- **Replication Factor (RF)** — number of partition copies (typically 3)
- **min.insync.replicas** — minimum replicas that must ACK a write for `acks=all`
- **Retention Period** — how long data stays on local storage (days)
- **Consumer Groups** — number of independent consumer groups reading the data

**Broker Hardware Assumptions:**
- **Disk Write** — sustained sequential write speed per disk (NVMe ~500 MB/s, SSD ~300 MB/s, HDD ~150 MB/s)
- **JBOD Disks / Broker** — independent disks per broker; each adds throughput linearly
- **Max Storage / Broker** — total usable storage per broker across all disks
- **NIC Bandwidth** — network interface capacity with presets (10G, 25G, 2x25G bonded) or custom
- **Target Utilization** — max utilization ceiling (50% recommended for rolling upgrade headroom)

**Tiered Storage:**
- **Enable/Disable** — toggle KIP-405 offloading to object storage
- **Hot Tier Retention** — hours of data kept on fast local NVMe (6–48h typical)

### Throughput

Calculates all data flow rates at peak:

| Metric | Formula |
|---|---|
| Ingress (steady) | `R × S` |
| Ingress (peak) | `Steady × Peak Multiplier` |
| Wire ingress (compressed) | `Peak / Compression Ratio` |
| Internal replication write | `Peak × RF` |
| Consumer egress | `Peak × Consumer Groups` |
| **Total broker I/O** | **`Peak × (RF + Consumer Groups)`** |

For multi-DC setups, also computes cross-DC replication bandwidth.

### Storage

Computes total disk requirements with a 1.35x headroom factor for segment overhead, compaction, and index files:

| Metric | Formula |
|---|---|
| Raw retention | `R × S × Retention_days × 86400` |
| Compressed on disk | `Raw / Compression Ratio` |
| With replication | `Compressed × RF` |
| **Total with headroom** | **`With_RF × 1.35`** |

When tiered storage is enabled, splits into:
- **Hot tier (local)** — computed from Hot Tier Hours instead of full retention
- **Cold tier (object storage)** — the remainder, with local disk savings percentage

### Brokers

Determines broker count by finding the most constraining resource:

| Constraint | Formula |
|---|---|
| Network-bound | `ceil(Total_IO / (NIC × Utilization))` |
| Disk write-bound | `ceil(Replication_Write / (Disk_Write × JBOD × Utilization))` |
| Storage-bound | `ceil(Local_Disk / (Max_Storage × Utilization))` |
| Minimum RF | `RF` (at least RF brokers required) |

**Final broker count** = `max(network, disk_write, storage, RF)`. The limiting constraint is highlighted.

Also provides per-broker hardware recommendations (CPU, RAM, JVM heap, disk layout, NIC) scaled to the ingress tier, and KRaft controller sizing (3 or 5 node quorum depending on cluster size).

### Schema Registry

Optional component sizing for Confluent Schema Registry:

| Metric | Formula |
|---|---|
| JVM Heap / instance | `max(256 MB, schemas × 0.5 MB)` |
| CPU / instance | `max(1, ceil(lookups / 10,000))` |
| RAM / instance | `ceil(heap_MB / 1024) + 1 GB` |

Configurable inputs: number of schemas, schema lookups/sec, and HA instance count. Totals are integrated into topology and summary tabs.

### REST Proxy

Optional component sizing for Confluent REST Proxy:

| Metric | Formula |
|---|---|
| CPU / instance | `max(2, ceil(requests / (instances × 2,500)))` |
| RAM / instance | `max(2, ceil(throughput_MB / 50) + 2) GB` |

Configurable inputs: HTTP request rate, average payload size, and instance count. REST Proxy is **stateful for consumption** — consumer group instances are pinned to a specific proxy, requiring sticky sessions. Producing is stateless. Totals are integrated into topology and summary tabs.

### Topology

Compares two multi-DC architectures when more than 1 datacenter is selected. Supports custom DC names and displays full mesh connectivity (including DC-1 to DC-3 links for 3-DC setups). Architecture diagrams with storage visualization (hot/cold tier bars) can be exported as PNG or SVG.

**Stretch Cluster:**
- Single logical Kafka cluster spanning DCs via `broker.rack`
- Synchronous replication built into RF — every `acks=all` produce waits for cross-DC ACK
- RPO = 0 (zero data loss), strong data consistency
- Trade-off: produce latency increases by inter-DC RTT
- Total brokers = same as single-DC count (replicas distributed across DCs)

**Multiple Clusters:**
- Independent Kafka cluster per DC, each sized for 100% failover capacity
- Asynchronous replication via MirrorMaker 2 or Confluent Cluster Linking
- Local produce latency (no cross-DC penalty)
- Trade-off: RPO > 0, eventual consistency, possible data divergence
- Total brokers = single-DC count × number of DCs
- Computes MirrorMaker 2 fleet sizing (workers, CPU, RAM) and cross-DC link bandwidth

Both topology sections include total CPU/RAM per DC accounting for all components (brokers, controllers, MM2, Schema Registry, REST Proxy). Includes a side-by-side comparison table and architecture diagrams.

### Partitions

Guidelines based on a 30 MB/s per-partition ceiling:

- **Minimum partitions** = `ceil(Peak_Ingress / 30 MB/s)`
- **Recommended** = `max(6, minimum)` — at least 6 for consumer parallelism
- **Per-broker limit** — ~4,000 partitions for low latency
- **Cluster-wide limit** — brokers × 4,000 (KRaft supports ~2M cluster-wide)

### Scenarios

Side-by-side comparison of 4 independent workload profiles. Each scenario has its own editable inputs (event rate, event size, peak multiplier, RF, retention, compression, consumer groups, DCs, utilization, disk write, JBOD count, NIC, max storage). All calculated results update independently. Default scenarios range from 10K to 10M events/sec.

### Monitoring

Reference table of key Kafka JMX/Prometheus metrics with warning and critical thresholds:
- Replication health (UnderReplicatedPartitions, IsrShrinksPerSec)
- Thread saturation (RequestHandler, NetworkProcessor idle percentages)
- Latency (Produce/Fetch p99, LogFlush p99)
- Resource utilization (CPU, disk, NIC percentages)
- Cluster health (ActiveControllerCount, OfflinePartitionsCount)

### Summary

Aggregated infrastructure view per datacenter and across all DCs:
- Broker and controller counts with per-node specs
- Schema Registry and REST Proxy instances (if enabled)
- MirrorMaker 2 fleet (if multi-cluster topology)
- Total CPU cores, RAM, local disk, and object storage (per DC and across all DCs)
- Cross-DC bandwidth requirements

Includes a collapsible **Sizing Computations Report** — a structured plain-text export of all calculations with formulas and actual values, ready to copy/paste into Word or PDF documents.

## Configuration Save / Load

Sizing configurations can be named, saved as shareable URLs, and restored on page load.

- **Name** — give your sizing configuration a descriptive name (e.g. "Production cluster Q3 2026")
- **Save as URL** — serializes all sizing inputs into an unsigned JWT (`alg: "none"`) stored in a `?config=` query parameter; the URL is copied to clipboard and can be shared or bookmarked
- **Load** — opening a URL with a `?config=` parameter automatically restores all inputs (workload, hardware, tiered storage, SR/RP, topology, DC names) and recomputes all results
- **Reset** — restores all inputs to factory defaults

The JWT payload contains a flat JSON object with all input field values, the topology mode, and DC names. No signature or server is required — the configuration is entirely client-side.

## Architecture Diagram Builder

A standalone diagram design tool that reuses the same visual components as the Sizing Calculator's Topology tab but with fully manual control over all parameters.

**DC Layout:**
- **Topology** — stretch cluster (single logical cluster) or multiple independent clusters
- **Number of Datacenters** — 1, 2, or 3
- **Custom DC Names** — user-defined names for each datacenter
- **Per-link Inter-DC Labels** — individual labels for each DC pair (e.g. different RTT values for DC-1↔DC-2 vs DC-2↔DC-3 vs DC-1↔DC-3)

**Components per DC:**
- Kafka Brokers, KRaft Controllers, Schema Registry, REST Proxy, MirrorMaker 2 Workers (set to 0 to hide any component)
- Hardware per Broker — configurable vCPU and RAM displayed in each DC box

**Storage Visualization:**
- Hidden, Local only, or Tiered (hot + cold) with custom labels and proportions

**Export:**
- PNG (2x retina) and SVG export using the same native rendering engine as topology diagrams

## Getting Started

### Run Locally

Open `index.html` in any modern browser. No server, build step, or dependencies required.

### Deploy to GitHub Pages

1. Push this repository to GitHub
2. Go to **Settings > Pages**
3. Set **Source** to **GitHub Actions**
4. The included workflow ([.github/workflows/deploy.yml](.github/workflows/deploy.yml)) will automatically deploy on every push to `main`

## Technology

- Pure HTML, CSS, and JavaScript — no frameworks, no dependencies
- CSS custom properties for consistent theming
- Google Fonts (Open Sans, JetBrains Mono) loaded from CDN
- Native SVG and Canvas 2D API for diagram export (no external libraries)
- Responsive grid layout for desktop and mobile

## License

[Apache License 2.0](LICENSE)
