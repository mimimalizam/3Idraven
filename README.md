# Central Logging & Monitoring Stack

Grafana + Loki + Promtail + Prometheus + Pushgateway — collects logs and metrics from **all** Docker containers on the server.

## What's in the stack

| Service     | Port | Description                                    |
|-------------|------|------------------------------------------------|
| Grafana     | 3000 | UI for logs and metrics (admin/admin)          |
| Loki        | 3100 | Log storage                                    |
| Promtail    | —    | Collects logs from all containers              |
| Prometheus  | 9090 | Metrics storage (scrapes Pushgateway)          |
| Pushgateway | 9091 | Receives metrics pushed from application jobs  |

## Deploy to server via SSH

### 1. Copy files to the server

```bash
scp -r . user@SERVER_IP:/opt/logging-stack
```

Or using rsync:

```bash
rsync -avz --exclude '.git' . user@SERVER_IP:/opt/logging-stack
```

### 2. Log in to the server

```bash
ssh user@SERVER_IP
```

### 3. Create the shared monitoring network

```bash
docker network create monitoring
```

This network allows other stacks (e.g. rezultati) to push metrics to Pushgateway.

### 4. Start the stack

```bash
cd /opt/logging-stack
docker compose up -d
```

### 5. Verify everything is running

```bash
docker compose ps
```

Grafana is available at `http://SERVER_IP:3000` (admin / admin).

## Adding logs from other stacks

Promtail automatically collects logs from **all** Docker containers on the host. All you need to do is add labels to containers in your other stacks:

```yaml
services:
  backend:
    image: my-backend
    labels:
      logging.stack: "stack-name"
      logging.app: "backend"
      logging.env: "production"
```

These labels are automatically converted into Loki labels for filtering.

## Queries in Grafana

```logql
# All logs from a stack
{stack="stack-name"}

# Specific service
{stack="stack-name", app="backend"}

# By Docker Compose project name (automatic)
{compose_project="directory-name"}

# Only error messages
{stack="stack-name"} |= "ERROR"

# All logs from the server
{job="docker"}
```

## Pushing metrics from application stacks

Application services can push metrics to Pushgateway (port `9091`). Prometheus scrapes Pushgateway every 15s and makes metrics available in Grafana.

Set `PUSHGATEWAY_URL` in your app's `.env`:

```bash
# If using a shared Docker network
PUSHGATEWAY_URL=pushgateway:9091

# If using host port (Linux)
PUSHGATEWAY_URL=172.17.0.1:9091
```

Example PromQL queries in Grafana (datasource: Prometheus):

```promql
# Seconds since last successful scheduler run
time() - scheduler_last_success_timestamp{stack="rezultati"}

# Number of changed endpoints per run
scheduler_endpoints_changed{stack="rezultati"}
```

## Updating configuration

When you change config locally, redeploy:

```bash
rsync -avz --exclude '.git' . user@SERVER_IP:/opt/logging-stack
ssh user@SERVER_IP "cd /opt/logging-stack && docker compose up -d"
```

## Retention

Loki retains logs for **31 days** (configured in `loki-config.yml` → `retention_period: 744h`).
