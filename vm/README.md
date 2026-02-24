# VM / Bare Metal (Linux) Setup

Monitor Typesense running directly on a Linux server using the New Relic Infrastructure Agent installed on the host.

> **Verified with:** nri-flex 1.17.2  
> ← [Back to main README](../README.md)

---

## Prerequisites

- Typesense 0.23+ running on the same host or accessible via network
- Linux server (Ubuntu 20.04+, Debian, RHEL/CentOS 7+, or Amazon Linux 2)
- New Relic account (free tier is sufficient)
- `sudo` access on the server

---

## Step 1 — Verify Typesense Endpoints Are Accessible

```bash
# From the server where the agent will be installed:
curl -s -H "X-TYPESENSE-API-KEY: YOUR_KEY" http://localhost:8108/health
# Expected: {"ok":true}

curl -s -H "X-TYPESENSE-API-KEY: YOUR_KEY" http://localhost:8108/stats.json | head -c 200
# Expected: {"search_requests_per_second":0,...}

curl -s -H "X-TYPESENSE-API-KEY: YOUR_KEY" http://localhost:8108/metrics.json | head -c 200
# Expected: {"system_cpu_active_percentage":...}
```

> If Typesense is on a different host, replace `localhost` with the actual IP/hostname.

---

## Step 2 — Install the New Relic Infrastructure Agent

Use the guided install or the manual steps below.

**Guided install (recommended):**  
Go to [New Relic → Add Data → Guided Install](https://one.newrelic.com/launcher/nr1-core.home) and follow the Linux Infrastructure Agent steps. `nri-flex` is bundled automatically.

**Manual install (Ubuntu/Debian):**

```bash
# Add New Relic GPG key and repo
curl -s https://download.newrelic.com/infrastructure_agent/gpg/newrelic-infra.gpg | \
  sudo gpg --dearmor -o /etc/apt/trusted.gpg.d/newrelic-infra.gpg

echo "deb https://download.newrelic.com/infrastructure_agent/linux/apt focal main" | \
  sudo tee /etc/apt/sources.list.d/newrelic-infra.list

sudo apt-get update
sudo apt-get install newrelic-infra -y
```

**Manual install (RHEL/CentOS/Amazon Linux):**

```bash
sudo curl -o /etc/yum.repos.d/newrelic-infra.repo \
  https://download.newrelic.com/infrastructure_agent/linux/yum/el/7/x86_64/newrelic-infra.repo

sudo yum -q makecache -y --disablerepo='*' --enablerepo='newrelic-infra'
sudo yum install newrelic-infra -y
```

**Set your License Key:**

```bash
echo "license_key: YOUR_NEW_RELIC_LICENSE_KEY" | \
  sudo tee /etc/newrelic-infra/newrelic-infra.yml
```

---

## Step 3 — Store the API Key Securely

Store the Typesense API key as an environment variable in the agent's environment file so it is not hardcoded in the config.

```bash
# Create the environment file for the Infrastructure Agent
sudo mkdir -p /etc/newrelic-infra
sudo tee /etc/newrelic-infra/agent-environment > /dev/null <<EOF
TYPESENSE_API_KEY=YOUR_TYPESENSE_API_KEY
EOF

sudo chmod 600 /etc/newrelic-infra/agent-environment
```

Then configure the systemd unit to load this file:

```bash
sudo mkdir -p /etc/systemd/system/newrelic-infra.service.d
sudo tee /etc/systemd/system/newrelic-infra.service.d/env.conf > /dev/null <<EOF
[Service]
EnvironmentFile=/etc/newrelic-infra/agent-environment
EOF

sudo systemctl daemon-reload
sudo systemctl restart newrelic-infra
```

Verify the variable is loaded:

```bash
sudo systemctl show newrelic-infra | grep TYPESENSE
# Expected: ... TYPESENSE_API_KEY=your-actual-key
```

---

## Step 4 — Deploy the Flex Config

Copy the Flex config to the integrations directory:

```bash
sudo cp flex/nri-flex-typesense.yml /etc/newrelic-infra/integrations.d/nri-flex-typesense.yml
```

If Typesense is on a **different host**, edit the file and replace `localhost` with the actual hostname or IP:

```bash
sudo sed -i 's/localhost/YOUR_TYPESENSE_HOST/g' \
  /etc/newrelic-infra/integrations.d/nri-flex-typesense.yml
```

Restart the agent to pick up the new config:

```bash
sudo systemctl restart newrelic-infra
sudo systemctl status newrelic-infra
```

---

## Step 5 — Verify with Dry Run

```bash
sudo -E /usr/bin/newrelic-infra -dry_run \
  -integration_config_path \
  /etc/newrelic-infra/integrations.d/nri-flex-typesense.yml
```

The `-E` flag passes the current environment (including `TYPESENSE_API_KEY`) to the command.

Expected output includes `TypesenseHealthSample`, `TypesenseStatsSample`, `TypesenseMetricsSample`, and `flexStatusSample` (normal internal metadata).

Data appears in New Relic within **1–2 minutes** after the agent restarts.

---

## Troubleshooting

```bash
# Is the agent running?
sudo systemctl status newrelic-infra

# Is the API key loaded?
sudo systemctl show newrelic-infra | grep TYPESENSE

# Is the config file in place?
ls /etc/newrelic-infra/integrations.d/ | grep typesense

# Can the agent reach Typesense?
curl -s -H "X-TYPESENSE-API-KEY: $TYPESENSE_API_KEY" http://localhost:8108/health

# Check agent logs
sudo journalctl -u newrelic-infra --since "10 minutes ago" | grep -i "typesense\|flex\|error"
```

| Symptom | Likely Cause | Fix |
|---|---|---|
| No data in New Relic | Agent not restarted after config change | `sudo systemctl restart newrelic-infra` |
| `TYPESENSE_API_KEY` not found | Environment file not loaded | Check `/etc/systemd/system/newrelic-infra.service.d/env.conf` |
| HTTP 401 on dry run | Wrong or expired API key | Re-check `agent-environment` file |
| Connection refused | Typesense not running or wrong host | `curl http://localhost:8108/health` |