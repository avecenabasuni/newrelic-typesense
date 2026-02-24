# New Relic - Typesense Integration

Monitor your self-hosted [Typesense](https://typesense.org/) search engine with [New Relic](https://newrelic.com/) using `nri-flex`, no custom exporters required.

> **Verified with:** nri-flex 1.17.2  
> **Typesense version:** 0.23+

## Choose Your Deployment Environment

| Environment | Guide | Use Case |
|---|---|---|
| ☸️ [Kubernetes / EKS](k8s/README.md) | `k8s/` | Typesense running in a K8s cluster |
| 🖥️ [VM / Bare Metal](vm/README.md) | `vm/` | Typesense on a Linux server |
| 🐳 [Docker / Docker Compose](docker/README.md) | `docker/` | Typesense in containers (incl. local dev) |

## How It Works

Typesense exposes three built-in monitoring endpoints. `nri-flex` scrapes them every 60 seconds and ships the data to New Relic as custom events.

```
Typesense                    New Relic Infrastructure Agent        New Relic Cloud
────────────────             ──────────────────────────────        ───────────────
/health          ─────────►  nri-flex (bundled with agent)  ────►  NRDB
/stats.json      ─────────►  HTTP scrape every 60s          ────►  Dashboard + Alerts
/metrics.json    ─────────►  auth via API key               ────►
```

| Endpoint | New Relic Event Type | Description |
|---|---|---|
| `/health` | `TypesenseHealthSample` | Node health status |
| `/stats.json` | `TypesenseStatsSample` | Request rate, latency, cache, overload |
| `/metrics.json` | `TypesenseMetricsSample` | CPU, memory, disk, network |

## Repository Structure

```
.
├── README.md                    ← You are here
├── k8s/                  ← Kubernetes / Amazon EKS setup
│   ├── README.md
│   ├── flex/
│   │   ├── nri-flex-typesense.yaml          # Single-node ConfigMap
│   │   └── nri-flex-typesense-cluster.yaml  # Multi-node cluster config
│   ├── helm/
│   │   └── values-example.yaml              # Annotated nri-bundle Helm values
│   └── secret/
│       └── secret-example.yaml              # Kubernetes Secret template
├── vm/                          ← VM / Bare Metal (Linux) setup
│   ├── README.md
│   └── flex/
│       └── nri-flex-typesense.yml           # Flex config for host agent
├── docker/                      ← Docker / Docker Compose setup
│   ├── README.md
│   ├── docker-compose.yml                   # Ready-to-use compose file
│   └── flex/
│       └── nri-flex-typesense.yml           # Flex config for containerized agent
└── dashboard/
    ├── dashboard-queries.md          ← All NRQL queries (shared across environments)
    └── typesense-dashboard.json      ← Importable New Relic dashboard (4 pages, 25+ widgets)
```

## NRQL Queries, Dashboard & Alerts

All NRQL queries and alert condition configs are shared across environments, the event types (`TypesenseHealthSample`, `TypesenseStatsSample`, `TypesenseMetricsSample`) are the same regardless of how the agent is deployed.

→ [`dashboard/dashboard-queries.md`](dashboard/dashboard-queries.md): all queries as plain NRQL  
→ [`dashboard/typesense-dashboard.json`](dashboard/typesense-dashboard.json): importable New Relic dashboard (4 pages, 25+ widgets)

**To import the dashboard:**
1. Open New Relic → Dashboards → Import dashboard
2. Replace all instances of `"accountId": 0` with your actual New Relic Account ID
3. Paste the JSON and import

## Field Reference

> ⚠️ The `ok` field in `TypesenseHealthSample` is a **String**, not a Boolean.  
> Use `WHERE ok = 'true'` with quotes in NRQL.

### TypesenseStatsSample

| Field | Unit | Description |
|---|---|---|
| `search_requests_per_second` | req/s | Search requests per second |
| `write_requests_per_second` | req/s | Write requests per second |
| `delete_requests_per_second` | req/s | Delete requests per second |
| `import_requests_per_second` | req/s | Import requests per second |
| `total_requests_per_second` | req/s | All request types combined |
| `overloaded_requests_per_second` | req/s | Requests rejected due to overload |
| `search_latency_ms` | ms | Average search latency |
| `write_latency_ms` | ms | Average write latency |
| `delete_latency_ms` | ms | Average delete latency |
| `import_latency_ms` | ms | Average import latency |
| `cache_hit_ratio` | 0.0–1.0 | Cache hit ratio |
| `pending_write_batches` | count | Write batches pending processing |

### TypesenseMetricsSample

| Field | Unit | Description |
|---|---|---|
| `system_cpu_active_percentage` | % | Average CPU across all cores |
| `system_cpu1_active_percentage` | % | Core 1 CPU usage |
| `system_cpu2_active_percentage` | % | Core 2 CPU usage |
| `system_cpu3_active_percentage` | % | Core 3 CPU usage |
| `system_cpu4_active_percentage` | % | Core 4 CPU usage |
| `system_memory_used_bytes` | bytes | System memory in use |
| `system_memory_total_bytes` | bytes | Total system memory |
| `system_disk_used_bytes` | bytes | Disk space used |
| `system_disk_total_bytes` | bytes | Total disk capacity |
| `system_network_received_bytes` | bytes | Network data received per interval |
| `system_network_sent_bytes` | bytes | Network data sent per interval |
| `typesense_memory_active_bytes` | bytes | Active memory used by Typesense |
| `typesense_memory_allocated_bytes` | bytes | Total memory allocated by Typesense |
| `typesense_memory_fragmentation_ratio` | ratio | Memory fragmentation; < 1.5 is healthy |

## References

- [Typesense API Reference](https://typesense.org/docs/latest/api/)
- [New Relic Flex Documentation](https://docs.newrelic.com/docs/infrastructure/host-integrations/host-integrations-list/flex-integration-tool-build-your-own-integration/)
- [New Relic NRQL Reference](https://docs.newrelic.com/docs/nrql/)
- [nri-bundle Helm Chart](https://github.com/newrelic/helm-charts/tree/master/charts/nri-bundle)

## License

MIT