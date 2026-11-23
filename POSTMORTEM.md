# Postmortem — checkout failures after payments 1.1.0 and a payments outage

Ruslanbek Tolametov · ShopLab · 2026-11-23 · blameless

## Summary

Payments 1.1.0 was deployed at 19:55 UTC; its new unionpay client fails on KZT, so every unionpay payment returned 500 and the shop answered 502 "payment failed" to those customers. At 20:10 payments was stopped for two minutes and **every** checkout failed with 502 "payments unreachable". The rollback to 1.0.0 at 20:17 ended the errors; 289 checkouts failed in total, about 2 % of the 30-day error budget.

## Impact

```promql
sum(increase(http_requests_total{job="shop",route="/api/checkout",status=~"5.."}[30m]))   # @ 20:25 → 288.8
sum(increase(http_requests_total{job="shop",route="/api/checkout"}[30m]))                 # @ 20:25 → 2002.6
```

- **Failed checkouts: 289 of 2003** (14.4 %) between 19:55 and 20:17 UTC (22 minutes of impact).
  - 145 unionpay payments rejected by payments 1.1.0 (19:55:12–20:16:56);
  - 143 checkouts of all card types during the outage, "payments unreachable" (20:10:11–20:12:01).

  ```logql
  sum by (upstream_error) (count_over_time({service="shop"} | json | path="/api/checkout" and status=502 and upstream_error!="" [30m]))  # 143
  sum(count_over_time({service="shop"} | json | path="/api/checkout" and status=502 and upstream_status="500" [30m]))                    # 145
  ```
- **Error budget:** 289 failed checkouts at ~66.8 checkouts/minute = **4.3 minutes of "fully down"** out of the 216-minute budget (99.5 % over 30 days) = **2.0 % of the budget** used in 22 minutes.

  ```promql
  sum(increase(http_requests_total{job="shop",route="/api/checkout",status=~"5.."}[30m]))
    / (sum(increase(http_requests_total{job="shop",route="/api/checkout"}[30m])) / 30) / 216    # 0.0200
  ```
- **Data exposure:** payments 1.1.0 wrote full 16-digit card numbers into its logs — 2 433 lines in Loki, only `version="1.1.0"`.

  ```logql
  sum by (version) (count_over_time({service="payments"} |~ `[0-9]{16}` | json [30m]))
  ```

## Timeline (UTC)

| Time | Event | Evidence |
|---|---|---|
| 19:55:00 | **Deploy** payments 1.1.0 | `date -u` + `PAYMENTS_VERSION=1.1.0 docker compose up -d payments`; `changes(process_start_time_seconds{job="payments"}[1h])` steps 0→1 at 19:55 |
| 19:55:12 | **First error**: `currency KZT not supported by the new unionpay client`, request `c81401bdf91f4f54`, shop 502 | `{service=~"shop\|payments"} \| json \| level="ERROR" and path!="/api/search"` |
| 19:58–19:59 | 5m burn rate passes 14.4 (page would need the 1h window too) | `shop:checkout_error_ratio:rate5m / 0.005` |
| 20:03:31 | **First alert** and **first useful alert**: `HighErrorRatio {job: payments}` "payments: 8.382% of requests return 5xx" | webhook log; `ALERTS{alertname="HighErrorRatio"}` |
| 20:04:56 | `CheckoutErrorBudgetSlowBurn` (ticket) "10.1x too fast" | webhook log; `ALERTS{alertname=~"Checkout.*"}` |
| 20:10:00 | **Outage**: payments stopped | `docker compose stop payments` |
| 20:10:11 | First "payments unreachable" (`Name or service not known`) | `{service="shop"} \| json \| upstream_error != ""` |
| 20:10:15 | `up{job="payments"}` = 0 | `up{job="payments"}` |
| 20:11:16 | `TargetDown {job: payments}` (page) | webhook log |
| 20:12:00 | payments started (still 1.1.0); `up` = 1 at 20:12:10 | `docker compose start payments`; `changes(...)` → 2 |
| 20:12:51 | `CheckoutErrorBudgetFastBurn` (page) "28.6x too fast" | webhook log |
| 20:15:56 | `HighErrorRatio {job: shop}` 12.22 % | webhook log |
| 20:16:16 | TargetDown resolved | webhook log |
| 20:16:56 | last unionpay 500 (1.1.0) | `{service="payments"} \| json \| msg!="request" and level="ERROR"` |
| 20:17:00 | **Rollback** to 1.0.0 | `docker compose up -d payments`; `changes(...)` → 3 |
| 20:20:56–20:23:31 | **Resolved**: HighErrorRatio shop, FastBurn, HighErrorRatio payments | webhook log `[RESOLVED]`; 5m burn rate 0 from 20:22 |
| 20:44:56 | `CheckoutErrorBudgetSlowBurn` resolved (the 30m window had to clear the outage) — **incident over** | webhook log `[RESOLVED]` |

## Detection

- **First and most useful:** `HighErrorRatio {job="payments"}` at 20:03:31, 8.5 minutes after the deploy. It named the service that causes the errors and the size (8.4 %), and its runbook leads to the card-type query.
- The SLO alerts confirmed the customer impact: slow burn (ticket) at 20:04:56; fast burn (page) only at 20:12:51, during the outage, because a 10 % error ratio is just above the 14.4× threshold and needs most of the 1h window (see Task 2).
- `TargetDown` at 20:11:16 explained the outage within 76 s.
- **Noise:** `HighErrorRatio {job="shop"}` at 20:15:56 was the same incident seen from the shop and came after payments was already back up; the FastBurn page arrived after TargetDown had already paged for the outage. With the old `ErrorsDetected` rule the same night produced ~40 notifications per 15 minutes (Task 3), and nobody read them.

## Root cause

1. **Payments 1.1.0 rejects unionpay in KZT.** Every 500 from payments is a unionpay payment, and the shop's 502s match them one to one:

   ```promql
   sum by (job, route, status) (rate(http_requests_total{status=~"5.."}[5m]))   # shop /api/checkout 502 + payments /api/pay 500
   ```
   ```logql
   sum by (card) (count_over_time({service="payments"} | json | status >= 500 [30m]))   # only card="unionpay"
   {service="payments"} | json | level="ERROR" | msg != "request"                       # currency KZT not supported by the new unionpay client
   sum by (version) (count_over_time({service="payments"} | json | status >= 500 [30m]))   # only version="1.1.0"
   ```
   The release went to 100 % of traffic at once with no test for unionpay/KZT and no canary.
2. **The outage:** with payments stopped, the shop could not resolve `payments` (`upstream_error: Name or service not known`), so every checkout failed regardless of card. There is one payments instance and no retry or fallback in the shop.
3. **The leak:** 1.1.0 adds `card_number` to the access log fields (`g.extra["card_number"]`), and nothing redacts it.

## What went well

- The new alerts were few and specific: 5 FIRING notifications for the whole night, each with a number and a runbook.
- The `request_id` passed from shop to payments let one LogQL query show the 502 and its cause together.
- Rollback was one command and the 5m burn rate was 0 five minutes later.

## What went badly

- 8.5 minutes from first error to first alert; the page-level SLO alert came only after the second failure (the outage).
- The release went to all users at once; nothing compared 1.1.0 with 1.0.0 before full rollout.
- Restarting a known-bad version (20:12 start was still 1.1.0) prolonged the unionpay errors; the rollback came 5 minutes later.
- Card numbers were written to Loki (retention 168 h) and nobody was alerted.
- After the restart Alloy shipped the payments container's earlier lines to Loki again, so metric LogQL queries over payments double-count that period (range queries de-duplicate). Counts in this report use the shop's logs and Prometheus.

## Action items

| # | Action | Type | Owner |
|---|---|---|---|
| 1 | Canary release for payments (week 12): 1.1.0 to 10 % of traffic, automatic rollback if the 5xx ratio of the canary > 1 % for 5 min | prevent | Ruslanbek Tolametov |
| 2 | Contract test for every card type × currency (unionpay + KZT) in CI before a payments release | prevent | Ruslanbek Tolametov |
| 3 | Redact card numbers in the payments logger (keep the last 4 digits) + a LogQL alert on `|~ "[0-9]{16}"`; delete the leaked lines from Loki | prevent / detect | Ruslanbek Tolametov |
| 4 | Alert on a new version's error ratio right after a deploy (`HighErrorRatio` with `for: 2m` during a release window) and annotate deploys in Grafana | detect | Ruslanbek Tolametov |
| 5 | Inhibit checkout SLO / shop `HighErrorRatio` while `TargetDown{job="payments"}` fires, so one page explains the outage | detect | Ruslanbek Tolametov |
| 6 | Run two payments replicas and add a short retry with timeout in the shop; rollback runbook: "roll back first, investigate later" | mitigate | Ruslanbek Tolametov |
