# Runbook: TargetDown

**Fires when:** Prometheus cannot scrape a target (`up == 0`) for 30 s. Severity `page`.

1. `up` in Prometheus, `docker compose ps` — is the container running?
2. `docker compose logs --tail 50 <service>` — crash, OOM, wrong config?
3. `docker compose start <service>` (or `up -d <service>`), wait for `up == 1`.
4. While payments is down every checkout fails with 502 "payments unreachable":
   `{service="shop"} | json | upstream_error != ""`. Expect the checkout SLO alerts too.
