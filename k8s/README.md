# Kubernetes Setup

Monitor Typesense running inside a Kubernetes cluster using `nri-flex` bundled with the New Relic Infrastructure Agent DaemonSet.

> **Verified with:** nri-flex 1.17.2 on Amazon EKS  
> ← [Back to main README](../README.md)

---

## Prerequisites

| Component | Minimum Version | Notes |
|---|---|---|
| Typesense | 0.23+ | Self-hosted in cluster |
| New Relic Account | Free tier | License Key required |
| Amazon EKS | v1.24+ | With `kubectl` access |
| `nri-bundle` Helm chart | Latest | Includes nri-flex automatically |
| Helm | 3.x | For installing / upgrading |

**Before you begin, collect:**
- New Relic License Key
- Typesense API key (read-only scope is sufficient)
- Namespace where Typesense is running
- `nri-bundle` Helm release name: `helm list -n newrelic`

---

## Step 1 — Verify Endpoint Access

```bash
kubectl run curl-test --image=curlimages/curl:latest --rm -it -n newrelic -- sh

# Inside the pod:
curl -s -H "X-TYPESENSE-API-KEY: YOUR_KEY" \
  http://typesense.<NAMESPACE>.svc.cluster.local:8108/health
# Expected: {"ok":true}
```

> HTTP 401 → API key lacks permissions. HTTP 000 → wrong DNS name or Typesense not running.

---

## Step 2 — Store the API Key as a Kubernetes Secret

```bash
kubectl create secret generic typesense-api-key \
  --from-literal=api-key=YOUR_TYPESENSE_API_KEY \
  -n newrelic

# Verify
kubectl get secret typesense-api-key -n newrelic
```

See [`k8s/secret-example.yaml`](k8s/secret-example.yaml) for a YAML template.

---

## Step 3 — Configure Helm values.yaml

Add to your existing `values.yaml` for `nri-bundle`. Replace `<NAMESPACE>` with the namespace where Typesense runs.

```yaml
newrelic-infrastructure:
  extraEnv:
    - name: TYPESENSE_API_KEY
      valueFrom:
        secretKeyRef:
          name: typesense-api-key
          key: api-key

  integrations_config:
    - name: nri-flex-typesense.yml
      data: |
        integrations:
          - name: nri-flex
            interval: 60s
            config:
              name: TypesenseMonitoring
              apis:
                - name: TypesenseHealth
                  url: http://typesense.<NAMESPACE>.svc.cluster.local:8108/health
                  headers:
                    X-TYPESENSE-API-KEY: ${TYPESENSE_API_KEY}
                - name: TypesenseStats
                  url: http://typesense.<NAMESPACE>.svc.cluster.local:8108/stats.json
                  headers:
                    X-TYPESENSE-API-KEY: ${TYPESENSE_API_KEY}
                - name: TypesenseMetrics
                  url: http://typesense.<NAMESPACE>.svc.cluster.local:8108/metrics.json
                  headers:
                    X-TYPESENSE-API-KEY: ${TYPESENSE_API_KEY}
```

See [`helm/values-example.yaml`](helm/values-example.yaml) for the fully annotated template.

**Prefer a standalone ConfigMap?** Use [`flex/nri-flex-typesense.yaml`](flex/nri-flex-typesense.yaml) and apply separately. Note: do not use both approaches simultaneously — this causes duplicate data.

---

## Step 4 — Apply and Verify

```bash
# Upgrade Helm release
helm upgrade <RELEASE_NAME> newrelic/nri-bundle -n newrelic -f values.yaml

# Wait for rollout
kubectl rollout status daemonset/newrelic-bundle-nrk8s-kubelet -n newrelic

# Confirm API key is injected
INFRA_POD=$(kubectl get pods -n newrelic \
  -l app.kubernetes.io/name=newrelic-infrastructure \
  -o jsonpath='{.items[0].metadata.name}')
kubectl exec -n newrelic $INFRA_POD -- env | grep TYPESENSE

# Watch logs
kubectl logs -n newrelic $INFRA_POD --since=5m | grep -i "typesense\|flex\|error"
```

Data appears in New Relic within **1–2 minutes**.

---

## Dry Run

Validate config without sending data to New Relic:

```bash
kubectl exec -n newrelic $INFRA_POD -- \
  /usr/bin/newrelic-infra -dry_run \
  -integration_config_path \
  /etc/newrelic-infra/integrations.d/nri-flex-typesense.yml
```

Expected output includes `TypesenseHealthSample`, `TypesenseStatsSample`, `TypesenseMetricsSample`, and `flexStatusSample` (the last one is normal internal metadata).

---

## Multi-Node Cluster

Use [`flex/nri-flex-typesense-cluster.yaml`](flex/nri-flex-typesense-cluster.yaml) for Typesense running as a StatefulSet. Each node gets a `node_id` attribute for per-node filtering in NRQL:

```sql
SELECT average(search_latency_ms)
FROM TypesenseStatsSample
FACET node_id
TIMESERIES AUTO SINCE 1 hour ago
```

---

## Troubleshooting

```bash
# Pod running?
kubectl get pods -n newrelic | grep newrelic-infrastructure

# API key injected?
kubectl exec -n newrelic $INFRA_POD -- env | grep TYPESENSE

# Flex config mounted?
kubectl exec -n newrelic $INFRA_POD -- \
  ls /etc/newrelic-infra/integrations.d/ | grep typesense

# Can agent reach Typesense?
kubectl exec -n newrelic $INFRA_POD -- \
  curl -s -H "X-TYPESENSE-API-KEY: $TYPESENSE_API_KEY" \
  http://typesense.<NAMESPACE>.svc.cluster.local:8108/health

# Error logs
kubectl logs -n newrelic $INFRA_POD --since=10m | grep -i "error\|fail"
```