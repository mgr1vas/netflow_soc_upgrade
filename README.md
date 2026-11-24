> **Academic Thesis Notice**  
> This repository contains the implementation and benchmarking suite of an undergraduate thesis by **Marios Grivas**, conducted under the supervision of **Christos Gogos** at University of Ioannina, dept Informatics & Telecommunications..
Project Overview
This thesis focuses on upgrading a university network flow monitoring pipeline from a file-based prototype to a real-time, database-backed Security Operations Center (SOC) architecture, followed by a comprehensive Before vs. After stress test.
Legacy Prototype (Before): `Router / softflowd` → `nfcapd` (UDP 9995, 5-min rotation) → `nfdump` + Python HTTP JSON bridge (`flow_bridge.py`) → `Grafana` (Infinity Plugin).
Modern Prototype (After): `Router / softflowd / NFStream` → `GoFlow2` (UDP 2055 / 6343) → `ClickHouse` (`flows_db.flows`, columnar OLAP storage) → `Grafana` (ClickHouse Data Source & SOC Dashboards).
---
Dashboard / Architecture Preview
![Project Preview](assets/uoi_logo.jpg)
Figure 1: Overview of the upgraded SOC monitoring pipeline and Grafana dashboard.
---
Repository Structure
```text
netflow_soc_upgrade/
├── README.md
├── .gitignore
├── .env.example
├── legacy_prototype/
│   ├── start_collector.sh
│   ├── flow_bridge.py
│   └── grafana/
│       └── grafana_infinity_dashboard.json
├── modern_prototype/
│   ├── docker-compose.yml
│   ├── clickhouse/
│   │   ├── clickhouse_init.sql
│   │   └── clickhouse_soc_queries.sql
│   └── grafana/
│       ├── provisioning/
│       │   ├── datasources/
│       │   │   └── grafana_clickhouse_datasource.yml
│       │   └── dashboards/
│       │       └── grafana_dashboards_config.yml
│       └── dashboards/
│           └── grafana_soc_overview.json
├── benchmarks/
│   ├── generators/
│   │   ├── replay_flows.sh
│   │   └── synthetic_netflow.py
│   ├── load_tests/
│   │   ├── test_legacy_api.sh
│   │   └── test_clickhouse.sh
│   ├── monitoring/
│   │   └── collect_metrics.sh
│   └── results/
│       ├── benchmark_ingestion_metrics.csv
│       ├── benchmark_query_latency.csv
│       ├── benchmark_system_resources.csv
│       ├── benchmark_storage_footprint.csv
│       └── plot_results.py
└── docs/
    ├── soc_dashboard_preview.png
    ├── architecture_before.png
    └── architecture_after.png
```
---
Getting Started
1. Environment Setup
Copy the template environment file and update it with your credentials:
```bash
cp .env.example .env
```
2. Running the Legacy Prototype (Before)
Start the `nfcapd` collector and the Python JSON bridge:
```bash
./legacy_prototype/start_collector.sh
python3 legacy_prototype/flow_bridge.py
```
3. Running the Modern Prototype (After)
Launch `ClickHouse`, `GoFlow2`, and `Grafana` via Docker Compose:
```bash
cd modern_prototype
docker compose up -d
```
Initialize the ClickHouse schema using `modern_prototype/clickhouse/clickhouse_init.sql`, then access Grafana at `http://localhost:3000`.
---
Before vs. After Stress Testing
The `benchmarks/` directory automates reproducible performance comparisons between `legacy_prototype` and `modern_prototype` across four core dimensions:
Ingestion Rate & UDP Drops: `benchmarks/results/benchmark_ingestion_metrics.csv`
Query Latency (p50 / p95 / p99): `benchmarks/results/benchmark_query_latency.csv`
System Resource Overhead (CPU, RAM, Disk I/O): `benchmarks/results/benchmark_system_resources.csv`
Storage Footprint & Compression: `benchmarks/results/benchmark_storage_footprint.csv`
Run `benchmarks/results/plot_results.py` after collecting your test runs to generate the comparison charts.
---
Credits
Author: [Your Name]
Thesis Supervisor: Christos Gogos
