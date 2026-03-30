# Central Logging Stack

Grafana + Loki + Promtail — collects logs from **all** Docker containers on the server.

## What's in the stack

| Service  | Port | Description                               |
|----------|------|-------------------------------------------|
| Grafana  | 3000 | Log viewer UI (admin/admin)               |
| Loki     | 3100 | Log storage                               |
| Promtail | —    | Collects logs from all containers         |

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

### 3. Start the stack

```bash
cd /opt/logging-stack
docker compose up -d
```

### 4. Verify everything is running

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

## Updating configuration

When you change config locally, redeploy:

```bash
rsync -avz --exclude '.git' . user@SERVER_IP:/opt/logging-stack
ssh user@SERVER_IP "cd /opt/logging-stack && docker compose up -d"
```

## Retention

Loki retains logs for **31 days** (configured in `loki-config.yml` → `retention_period: 744h`).
