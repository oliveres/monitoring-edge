# Edge Monitoring Stack (Unified)

This repository provides a single GitOps-friendly stack for every edge host. The baseline deployment gathers host and container metrics plus Docker logs; optional exporters for PostgreSQL/TimescaleDB, PgBouncer, and Caddy metrics are toggled via Docker Compose profiles in the same `docker-compose.yml`.

## Core Components

- **Prometheus Agent** – performs local scrapes, keeps zero retention, and remote-writes to the central Prometheus.
- **Promtail** – ships Docker container logs to the central Loki.
- **cAdvisor** – exposes per-container CPU, memory, and I/O metrics.
- **Node Exporter** – publishes host-level metrics (CPU, memory, disks, network, load).

## Optional Exporters (Docker Compose Profiles)

| Profile     | Description                                                                 |
|-------------|-----------------------------------------------------------------------------|
| `caddy`     | Enables the Prometheus scrape job for an external Caddy instance (no container is launched by this stack). |
| `postgres`  | Starts `postgres-exporter` and enables PostgreSQL / TimescaleDB scraping.   |
| `pgbouncer` | Starts `pgbouncer-exporter` for PgBouncer connection-pool metrics.          |

Activate profiles with e.g. `COMPOSE_PROFILES=postgres docker compose up -d`. In Portainer set **Environment variables → COMPOSE_PROFILES** (for example `postgres,pgbouncer`).

## Requirements

- Docker 20.10+ and Docker Compose v2
- Reachable central Prometheus/Loki endpoints (via VPC or HTTPS)
- For database exporters: an external Docker network (`POSTGRES_NETWORK`) that reaches PostgreSQL/PgBouncer

## Deployment

### Portainer GitOps

1. Navigate to **Stacks → Add stack → Git repository**.
2. Repository URL: `https://github.com/YOUR-USERNAME/monitoring-edge`.
3. Provide environment variables as documented in `.env.example`.
4. If you need exporters, add `COMPOSE_PROFILES` and the relevant DSNs.
5. Deploy and optionally enable automatic Git sync.

### Manual `docker compose`

```bash
git clone https://github.com/YOUR-USERNAME/monitoring-edge.git
cd monitoring-edge
cp .env.example .env
# customise .env for this host
docker compose up -d                         # baseline monitoring only
COMPOSE_PROFILES=caddy docker compose up -d           # enable Caddy scrape job
COMPOSE_PROFILES=postgres docker compose up -d        # add PostgreSQL exporter
COMPOSE_PROFILES=postgres,pgbouncer docker compose up -d  # PostgreSQL + PgBouncer
```

## Environment Configuration

See `.env.example` for the authoritative list. Key entries:

- `HOSTNAME` – unique identifier for the host (propagated as `host` label).
- `CENTRAL_PROMETHEUS_URL`, `CENTRAL_LOKI_URL` – endpoints of the central stack.
- `BASIC_AUTH_USER`, `BASIC_AUTH_PASSWORD` – required only when using HTTPS with basic auth.
- `CADDY_METRICS_TARGET` – target for scraping Caddy metrics when the `caddy` profile is enabled.

### PostgreSQL / TimescaleDB (`postgres` profile)

- `POSTGRES_DATA_SOURCE_NAME` – DSN consumed by `postgres-exporter`.
- `POSTGRES_EXPORTER_TARGET` – usually `postgres-exporter:9187`; leave blank to disable the job.
- `POSTGRES_NETWORK` – external Docker network that reaches the database or PgBouncer.

### PgBouncer (`pgbouncer` profile)

- `PGBOUNCER_EXPORTER_CONN_STRING` – connection string to PgBouncer’s stats database.
- `PGBOUNCER_EXPORTER_TARGET` – metrics endpoint, defaults to `pgbouncer-exporter:9127` when using the bundled service.

## Prometheus Scrape Behavior

- Node Exporter rewrites both `instance` and `nodename` labels to `HOSTNAME`, so community dashboards (e.g. *Node Exporter Full*) work without manual tweaks.
- Scrape jobs for Caddy, PostgreSQL, and PgBouncer activate only when their respective target value is non-empty; relabel rules drop empty targets to avoid failing scrapes.

## Caddy Metrics (`caddy` Profile)

The edge stack does **not** run a Caddy container; the profile is a convention indicating that Prometheus should scrape an external Caddy instance already running on the host.

1. Update the host’s Caddyfile (for example `/etc/caddy/Caddyfile`) and add:

   ```caddy
   {
       admin 0.0.0.0:2019
       metrics
   }
   ```

2. Restart Caddy so it exposes `/metrics` on port 2019.
3. Set `COMPOSE_PROFILES=caddy` and `CADDY_METRICS_TARGET` (e.g. `localhost:2019`) in `.env`.
4. Reapply the stack (`docker compose up -d`). The Prometheus job `caddy` becomes active and metrics surface in Grafana, e.g. `caddy_admin_http_requests_total{host="edge-host-1"}`.

## Verification Checklist

1. `docker ps | grep monitoring-` – confirms the selected services are running.
2. Central Grafana Explore:
   - `up{host="edge-host-1"}` – baseline health.
   - `caddy_admin_http_requests_total{host="edge-host-1"}` – Caddy scrape (when enabled).
   - `pg_up{job="postgres",host="edge-host-1"}` – PostgreSQL exporter health.
3. Inspect logs: `docker logs monitoring-prometheus` and `docker logs monitoring-promtail`.

## Migrating from the Legacy Edge Repositories

1. Stop existing `monitoring-edge-basic` and `monitoring-edge-postgres` stacks (Portainer or CLI).
2. Deploy the new `monitoring-edge` stack with the same `HOSTNAME` value.
3. Copy required environment variables (`POSTGRES_DATA_SOURCE_NAME`, etc.).
4. Verify in Grafana that metrics still appear under the original `host` label.

## Additional Notes

- Leave the PgBouncer profile disabled (and env vars empty) if PgBouncer is not in use.
- Historical Prometheus series with hexadecimal `nodename` values may persist until retention expires; use the admin API to delete them sooner if needed.
- Compose profiles make it straightforward to introduce further optional exporters (e.g. Redis) without creating another repository—add the service and guard it with a new profile.
