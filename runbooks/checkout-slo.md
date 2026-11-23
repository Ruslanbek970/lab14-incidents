# Runbook: CheckoutErrorBudgetFastBurn / CheckoutErrorBudgetSlowBurn

**SLO:** 99.5 % of `POST /api/checkout` requests succeed over 30 days.
Error budget = 0.5 % = 216 minutes of full outage per 30 days.

| Alert | Condition | Severity | Meaning |
|---|---|---|---|
| CheckoutErrorBudgetFastBurn | burn rate > 14.4 over 1h **and** 5m | page | 2 % of the monthly budget gone in 1 hour; at this speed the budget lasts ~2 days |
| CheckoutErrorBudgetSlowBurn | burn rate > 6 over 6h **and** 30m | ticket | 5 % of the budget gone in 6 hours; budget lasts ~5 days |

burn rate = error ratio / 0.005

## 1. Confirm (Prometheus)

```promql
shop:checkout_error_ratio:rate5m / 0.005          # current burn rate
sum by (job, route, status) (rate(http_requests_total{status=~"5.."}[5m]))
```

- shop `/api/checkout` 502 + payments `/api/pay` 500 → payments returns errors.
- shop 502 only, payments shows nothing / `up{job="payments"} == 0` → payments is down or unreachable (see `TargetDown`).

## 2. Find what fails (Loki)

```logql
{service="shop"} | json | path="/api/checkout" and status>=500
sum by (card) (count_over_time({service="payments"} | json | status>=500 [10m]))
{service="payments"} | json | level="ERROR" | msg!="request"
```

Take one `request_id` and follow it: `{service=~"shop|payments"} |= "<request_id>"`.

## 3. Was there a change?

```promql
changes(process_start_time_seconds{job="payments"}[1h])
```

Check the deploy log (who deployed what and when) and `version` in the payments logs.

## 4. Mitigate

- A new payments release → roll back: `docker compose up -d payments` (back to 1.0.0).
- payments down → `docker compose start payments`, check `docker compose logs payments`.
- Watch `shop:checkout_error_ratio:rate5m` go back to 0; the alert resolves when the short window is under the threshold.

## 5. After

Write down the times (UTC) and the failed checkouts:
`sum(increase(http_requests_total{job="shop",route="/api/checkout",status=~"5.."}[1h]))`, then a blameless postmortem.
