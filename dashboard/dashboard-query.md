# NRQL Dashboard Queries — Typesense + New Relic

All queries verified via dry run on nri-flex 1.17.2.  
Field names are case-sensitive. See the Field Reference in README.md.

**How to add queries to a dashboard:**  
New Relic → Dashboards → Create Dashboard → Add Widget → NRQL Query

---

## Request Monitoring

### Search Request Rate
```sql
SELECT average(search_requests_per_second) AS 'Search Req/sec'
FROM TypesenseStatsSample
TIMESERIES AUTO
SINCE 1 hour ago
```
*Widget: Line Chart*

---

### Request Breakdown per Operation
```sql
SELECT
  average(search_requests_per_second) AS 'Search',
  average(write_requests_per_second)  AS 'Write',
  average(delete_requests_per_second) AS 'Delete',
  average(import_requests_per_second) AS 'Import',
  average(total_requests_per_second)  AS 'Total'
FROM TypesenseStatsSample
TIMESERIES AUTO
SINCE 1 hour ago
```
*Widget: Area Chart*

---

### Overloaded Requests
```sql
SELECT sum(overloaded_requests_per_second) AS 'Overloaded Req/sec'
FROM TypesenseStatsSample
TIMESERIES AUTO
SINCE 1 hour ago
```
*Widget: Line Chart — should always be 0 in a healthy environment*

---

### Pending Write Batches
```sql
SELECT average(pending_write_batches) AS 'Pending Write Batches'
FROM TypesenseStatsSample
TIMESERIES AUTO
SINCE 1 hour ago
```
*Widget: Line Chart — a steadily increasing value means write throughput exceeds capacity*

---

## Latency

### Search & Write Latency
```sql
SELECT
  average(search_latency_ms)  AS 'Search (ms)',
  average(write_latency_ms)   AS 'Write (ms)',
  average(delete_latency_ms)  AS 'Delete (ms)',
  average(import_latency_ms)  AS 'Import (ms)'
FROM TypesenseStatsSample
TIMESERIES AUTO
SINCE 1 hour ago
```
*Widget: Line Chart*

---

### Cache Hit Ratio
```sql
SELECT average(cache_hit_ratio) AS 'Cache Hit Ratio'
FROM TypesenseStatsSample
TIMESERIES AUTO
SINCE 1 hour ago
```
*Widget: Line Chart — add thresholds at 0.8 (warning) and 0.5 (critical)*

---

## CPU

### CPU Usage per Core
```sql
SELECT
  average(system_cpu_active_percentage)  AS 'All Cores (%)',
  average(system_cpu1_active_percentage) AS 'Core 1 (%)',
  average(system_cpu2_active_percentage) AS 'Core 2 (%)',
  average(system_cpu3_active_percentage) AS 'Core 3 (%)',
  average(system_cpu4_active_percentage) AS 'Core 4 (%)'
FROM TypesenseMetricsSample
TIMESERIES AUTO
SINCE 1 hour ago
```
*Widget: Line Chart*  
*Note: `system_cpuN` fields only exist for cores present on the server*

---

## Memory

### Memory Usage (MB)
```sql
SELECT
  average(typesense_memory_active_bytes)    / 1e6 AS 'Typesense Active (MB)',
  average(typesense_memory_allocated_bytes) / 1e6 AS 'Typesense Allocated (MB)',
  average(system_memory_used_bytes)          / 1e6 AS 'System Used (MB)',
  average(system_memory_total_bytes)         / 1e6 AS 'System Total (MB)'
FROM TypesenseMetricsSample
TIMESERIES AUTO
SINCE 1 hour ago
```
*Widget: Area Chart*

---

### Memory Fragmentation Ratio
```sql
SELECT average(typesense_memory_fragmentation_ratio) AS 'Fragmentation Ratio'
FROM TypesenseMetricsSample
TIMESERIES AUTO
SINCE 1 hour ago
```
*Widget: Line Chart — values consistently above 1.5 warrant investigation*

---

## Storage & Network

### Disk Usage (GB)
```sql
SELECT
  latest(system_disk_used_bytes)  / 1e9 AS 'Disk Used (GB)',
  latest(system_disk_total_bytes) / 1e9 AS 'Disk Total (GB)'
FROM TypesenseMetricsSample
SINCE 5 minutes ago
```
*Widget: Billboard*

---

### Network Throughput (KB/s)
```sql
SELECT
  rate(sum(system_network_received_bytes), 1 second) / 1024 AS 'Inbound (KB/s)',
  rate(sum(system_network_sent_bytes),     1 second) / 1024 AS 'Outbound (KB/s)'
FROM TypesenseMetricsSample
TIMESERIES AUTO
SINCE 1 hour ago
```
*Widget: Line Chart*

---

## Health

### Node Health Status

> ⚠️ `ok` is a **String** field. Use `WHERE ok = 'true'` with quotes — not `WHERE ok = true` or `WHERE ok = 1`.

```sql
SELECT
  filter(count(*), WHERE ok = 'true')  AS 'Healthy Checks',
  filter(count(*), WHERE ok = 'false') AS 'Failed Checks'
FROM TypesenseHealthSample
TIMESERIES AUTO
SINCE 1 hour ago
```
*Widget: Billboard — alert if Failed Checks > 0*

---

## Multi-Node Cluster Queries

### Compare Latency Across Nodes
```sql
SELECT average(search_latency_ms)
FROM TypesenseStatsSample
FACET node_id
TIMESERIES AUTO
SINCE 1 hour ago
```

### Health Status of All Nodes
```sql
SELECT latest(ok)
FROM TypesenseHealthSample
FACET node_id
SINCE 5 minutes ago
```

### CPU Usage Filtered to a Single Node
```sql
SELECT average(system_cpu_active_percentage)
FROM TypesenseMetricsSample
WHERE node_id = 'node0'
TIMESERIES AUTO
SINCE 1 hour ago
```

### Verify All Event Types Are Flowing
```sql
SELECT count(*)
FROM TypesenseHealthSample, TypesenseStatsSample, TypesenseMetricsSample
SINCE 10 minutes ago
FACET eventType()
```