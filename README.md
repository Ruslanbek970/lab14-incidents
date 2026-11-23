# lab14-incidents — ShopLab observability (Lab 14, Variant A)

Ruslanbek Tolametov

Two Flask services (`shop`, `payments`), a traffic generator, Prometheus, Alertmanager (+ a webhook
that prints notifications), Loki, Alloy and Grafana, all in Docker Compose.

**Variant:** A — Task 1 slow search, Task 2 checkout SLO + burn-rate alerts, Task 3 noisy alert,
Task 4 request_id across two services, Task 5 dashboard as code, plus the Final Challenge.

## Start

```bash
docker compose up -d --build
docker compose ps                 # 9 services, all Up
docker compose logs -f webhook    # notifications
```

| UI | URL |
|---|---|
| Prometheus | http://localhost:9090 |
| Alertmanager | http://localhost:9093 |
| Grafana (no login, local only) | http://localhost:3000 → Dashboards → lab14 → **ShopLab — RED** |
| Loki API | http://localhost:3100 |

## Release / rollback of payments

```bash
PAYMENTS_VERSION=1.1.0 docker compose up -d payments   # deploy
docker compose up -d payments                          # roll back to 1.0.0
date -u                                                # write down the time
```

## Check and reload the config

```bash
docker run --rm -v "$PWD/monitoring:/m" --entrypoint promtool prom/prometheus:v3.16.0 check rules /m/rules.yml
docker run --rm -v "$PWD/monitoring:/m" --entrypoint amtool prom/alertmanager:v0.35.0 check-config /m/alertmanager.yml
curl -X POST localhost:9090/-/reload
curl -X POST localhost:9093/-/reload
```

## Files

| Path | What |
|---|---|
| `app/` | `app.py` (shop + payments), `traffic.py`, Dockerfile |
| `monitoring/rules.yml` | final rules: `TargetDown`, checkout SLO recording rules (5m/30m/1h/6h), `CheckoutErrorBudgetFastBurn`, `CheckoutErrorBudgetSlowBurn`, `HighErrorRatio` (replaces the starter `ErrorsDetected`) |
| `monitoring/alertmanager.yml` | final routing: `group_by: [alertname, job]`, `group_wait: 30s`, `repeat_interval: 4h` |
| `dashboards/shoplab.json` | Grafana dashboard **ShopLab — RED**, provisioned on start |
| `runbooks/` | one runbook per alert |
| `REPORT.md` | all tasks: queries, commands, results, answers |
| `POSTMORTEM.md` | Final Challenge postmortem |

## Stop

```bash
docker compose down -v
docker image rm shoplab-app:1
```
