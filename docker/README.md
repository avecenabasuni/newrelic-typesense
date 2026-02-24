# Docker / Docker Compose Setup

Monitor Typesense running in Docker using the New Relic Infrastructure Agent container. This setup works for both local development and production Docker environments.

> **Verified with:** nri-flex 1.17.2  
> ← [Back to main README](../README.md)

---

## Prerequisites

- Docker Engine 20.10+
- Docker Compose v2 (or `docker-compose` v1.29+)
- New Relic account (free tier is sufficient) and a License Key
- Typesense running as a Docker container (or accessible via network)

---

## Step 1 — Verify Typesense Is Accessible

```bash
# From the host machine:
curl -s -H "X-TYPESENSE-API-KEY: YOUR_KEY" http://localhost:8108/health
# Expected: {"ok":true}
```

---

## Step 2 — Configure Environment Variables

Create a `.env` file in the same directory as your `docker-compose.yml`. **Do not commit this file.**

```bash
cat > .env <<EOF
NEW_RELIC_LICENSE_KEY=YOUR_NEW_RELIC_LICENSE_KEY
TYPESENSE_API_KEY=YOUR_TYPESENSE_API_KEY
EOF
```

Verify `.env` is in your `.gitignore`:

```bash
grep '.env' .gitignore || echo '.env' >> .gitignore
```

---

## Step 3 — Deploy with Docker Compose

The [`docker-compose.yml`](docker-compose.yml) in this directory includes both Typesense and the New Relic Infrastructure Agent. If you already have a running Typesense container, see the **Adding to an existing Compose file** section below.

```bash
# Start everything
docker compose --env-file .env up -d

# Check container status
docker compose ps

# Watch Infrastructure Agent logs
docker compose logs -f newrelic-infra
```

Data appears in New Relic within **1–2 minutes**.

---

## Step 4 — Verify with Dry Run

```bash
docker compose exec newrelic-infra \
  /usr/bin/newrelic-infra -dry_run \
  -integration_config_path \
  /etc/newrelic-infra/integrations.d/nri-flex-typesense.yml
```

Expected output includes `TypesenseHealthSample`, `TypesenseStatsSample`, `TypesenseMetricsSample`, and `flexStatusSample` (normal internal metadata).

---

## Adding to an Existing Compose File

If you already have a `docker-compose.yml` with Typesense, add the `newrelic-infra` service to it. Replace `typesense` with the actual service name of your Typesense container.

```yaml
services:
  newrelic-infra:
    image: newrelic/infrastructure:latest
    container_name: newrelic-infra
    network_mode: host          # Required for full host metric collection
    pid: host                  # Required for process-level metrics
    privileged: true           # Required for infrastructure metrics
    cap_add:
      - SYS_PTRACE
    environment:
      - NRIA_LICENSE_KEY=${NEW_RELIC_LICENSE_KEY}
      - TYPESENSE_API_KEY=${TYPESENSE_API_KEY}
      - NRIA_DISPLAY_NAME=typesense-host
    volumes:
      - /:/host:ro
      - /var/run/docker.sock:/var/run/docker.sock
      - ./flex/nri-flex-typesense.yml:/etc/newrelic-infra/integrations.d/nri-flex-typesense.yml:ro
    restart: unless-stopped
```

> **Note:** When using `network_mode: host`, the Flex config connects to Typesense via `localhost:8108`. If you prefer an isolated network (without `network_mode: host`), use the Typesense service name as the hostname instead (e.g., `http://typesense:8108`). In that case, remove `network_mode: host` and add the agent to the same Docker network as Typesense.

---

## Local Development Notes

For local dev where you just want metrics without full host-level monitoring, a lighter setup works too:

```bash
# Run the agent one-off for a dry run check (no persistent container)
docker run --rm \
  -e NRIA_LICENSE_KEY=YOUR_LICENSE_KEY \
  -e TYPESENSE_API_KEY=YOUR_API_KEY \
  -v $(pwd)/flex/nri-flex-typesense.yml:/etc/newrelic-infra/integrations.d/nri-flex-typesense.yml:ro \
  --network host \
  newrelic/infrastructure:latest \
  /usr/bin/newrelic-infra -dry_run \
  -integration_config_path /etc/newrelic-infra/integrations.d/nri-flex-typesense.yml
```

---

## Troubleshooting

```bash
# Is the container running?
docker compose ps

# Is the API key available inside the container?
docker compose exec newrelic-infra env | grep TYPESENSE

# Is the Flex config mounted?
docker compose exec newrelic-infra \
  ls /etc/newrelic-infra/integrations.d/ | grep typesense

# Can the agent reach Typesense?
docker compose exec newrelic-infra \
  curl -s -H "X-TYPESENSE-API-KEY: $TYPESENSE_API_KEY" http://localhost:8108/health

# Check logs for errors
docker compose logs newrelic-infra | grep -i "error\|fail\|typesense"
```

| Symptom | Likely Cause | Fix |
|---|---|---|
| Container exits immediately | `NRIA_LICENSE_KEY` not set | Check `.env` file and `docker compose --env-file .env up` |
| No data in New Relic | Agent can't reach Typesense | Verify network mode and Typesense URL in flex config |
| HTTP 401 on dry run | Wrong API key | Check `TYPESENSE_API_KEY` value in `.env` |
| Connection refused | Wrong hostname/port | Use `localhost:8108` with `network_mode: host`, or service name without it |
| `flexStatusSample` only, no other events | Endpoints returning errors | Run dry run and check full output for HTTP errors |