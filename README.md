# netflow_soc_upgrade

Upgrade of the NetFlow monitoring pipeline from a legacy `nfcapd + nfdump + Python bridge` prototype to a modern `GoFlow2 + ClickHouse + Grafana SOC` architecture, including a Before vs. After stress-testing suite.

## Repository Structure

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
    └── .gitkeep
```
