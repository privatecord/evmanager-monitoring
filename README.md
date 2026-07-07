# EVManager Monitoring

Monitoring stack for the EVManager project.

## Included

- Grafana dashboard for the EVManager overview
- Prometheus metric scraping and alert rules
- Alertmanager for notifications
- Loki and Promtail for logs
- cAdvisor for Docker container metrics
- node-exporter for host metrics

## Dashboard sections

- Overview: requests per second, active sessions, running containers and 5xx error rate
- System load: CPU, RAM and host usage
- Traffic: HTTP status codes, request latency and routes
- Database: DB query rate and DB query latency
- Logs: recent EVManager container logs

## Start locally

```bash
docker compose up -d
```

Grafana: http://localhost:3000

Default login:

```txt
admin / evmanager
```

Change the password before production usage.

## Required EVManager app metrics

Your app should expose Prometheus metrics at `/metrics`.

Expected metric names:

```txt
http_requests_total{route="/",method="GET",status="200"}
http_request_duration_seconds_bucket{route="/",method="GET",le="0.5"}
db_query_duration_seconds_bucket{operation="select",table="events",le="0.2"}
db_queries_total{operation="select",table="events"}
active_sessions
```

Infrastructure panels work through cAdvisor and node-exporter. App-specific panels become useful once the EVManager app exports these metrics.

## Alert behavior

Container-down alerts tolerate rebuilds/restarts up to 2 minutes:

```promql
absent(container_last_seen{name=~"evmanager.*"}) or time() - container_last_seen{name=~"evmanager.*"} > 120
```

## Dokploy

Use this repo as a Compose application in Dokploy. Adjust the `evmanager-app:3000` Prometheus target to the real service/container name if needed.
