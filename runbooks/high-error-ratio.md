# Runbook: HighErrorRatio

**Fires when:** more than 5 % of a service's requests return 5xx for 5 minutes, and the service gets
at least 0.1 req/s (no page for one error at night). Severity `page`.

## 1. Which service, which route

```promql
sum by (job, route, status) (rate(http_requests_total{status=~"5.."}[5m]))
sum by (job) (rate(http_requests_total{status=~"5.."}[5m])) / sum by (job) (rate(http_requests_total[5m]))
```

shop 502 on `/api/checkout` means the error comes from payments; shop 500 on its own routes means shop.

## 2. What the failing requests have in common

```logql
{service="<job>"} | json | status>=500
sum by (card) (count_over_time({service="payments"} | json | status>=500 [10m]))
{service="<job>"} | json | level="ERROR" | msg!="request"
```

## 3. Mitigate

Recent deploy (`changes(process_start_time_seconds{job="<job>"}[1h]) > 0`, `version` in logs) → roll back.
Otherwise escalate to the service owner with the query results and one `request_id`.
