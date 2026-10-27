# Postmortem: incident drill: catalogue unavailable

**Date:** 26 October 2026 · **Type:** planned incident drill · **Status:** resolved · **Format:** blameless

## Summary
As a drill, the catalogue deployment was scaled to **0 replicas** to simulate a crashed service. About **40% of all
requests** to the shop failed with **HTTP 503**, and shoppers could not browse products, so cart and checkout traffic
fell to zero. The drill also found a real gap: **the availability SLO did not detect the outage** at first, because it
was measured per backend service and Traefik counts nothing per service when a service has no pods. After the SLI was
changed to front-door metrics, the alert fired, the service was restored with one command, and the alert resolved.

## Impact
- **Users:** product listings and product pages failed (503); with no catalogue, users could not add to cart or pay.
- **Size:** front-door error ratio about **40%** for the whole outage (the SLO allows 1%).
- **Duration:** about **4.5 hours** (10:44 to ~15:20 UTC). Most of this was because the drill was left running while
  the operator was away (see below). Once started, the fix itself took under 2 minutes.
- **Error budget:** the 30-day budget is about 7.2 hours of full outage. 4.5 hours at 40% failure used roughly a
  quarter of it in a single afternoon.

## Timeline (UTC)
| Time | Event |
|---|---|
| 10:43 | Baseline checked: front-door error ratio 0.08%. All services healthy. |
| 10:44 | **Break:** `kubectl scale deploy catalogue -n robot-shop --replicas=0`. |
| 10:46 | Dashboard: catalogue, cart and payment request rates drop to 0. Users get 503 on `/api/catalogue/...`. |
| 10:48 | **No alert, SLI reads 0%.** Investigation: Traefik's per-service metric has nothing to count when a service has no pods; Traefik answers 503 itself. Its *entrypoint* metric shows 144 × 503 in 2 minutes. |
| 10:56 | SLI changed to `traefik_entrypoint_requests_total` (all requests reaching the shop). SLI now reads ~40%. |
| 10:59 | **Alert `RobotShopAvailabilitySLOBurn` FIRING** in Slack `#alerts`: "39.1% of requests are failing", with the runbook link. |
| 10:59 to 15:15 | Drill left unattended; outage continues at ~40% errors. |
| 15:04 | Alertmanager re-sends FIRING ("40.54% of requests are failing"), 4 hours after the first, per `repeat_interval`. |
| 15:17 | **Mitigation:** `kubectl scale deploy catalogue -n robot-shop --replicas=1`. Catalogue `1/1 Running`. |
| 15:24 | **Alert RESOLVED** in Slack. |
| ~15:28 | Front-door error ratio back under 1% (0.23%). |

## Root cause
The catalogue deployment had **zero running replicas** (deliberately, for the drill), so Traefik had no backend for
`/api/catalogue/...` and returned 503 for every such request.

## Contributing factor found by the drill
The availability SLI was built on `traefik_service_requests_total`, which only counts requests that reach a backend
service. When a service has no pods, its requests are rejected by Traefik before reaching any service, so they were
**invisible to the SLO**. A total outage of one service, the most serious kind of failure, could not page anyone.

## What went well
- The RED dashboard showed the outage immediately (three services dropping to zero).
- Comparing the per-service and entrypoint metrics found the blind spot within minutes.
- Mitigation was one command, as described in the runbook, and the alert resolved by itself.
- Alertmanager's repeat notification kept the incident visible while it was unresolved.

## What went badly
- The SLO missed a complete service outage until it was redesigned during the drill.
- The outage lasted hours because no one acted after the alert fired. In a real service this needs on-call
  coverage, not one person.
- The per-service "Error rate (%)" dashboard panel also showed 0% during the outage, for the same reason.

## Actions
| Action | Status |
|---|---|
| Measure the availability SLI at the Traefik entrypoint, so unroutable requests count | ✅ Done (`observability/slo-rules.yaml`) |
| Add an "Availability SLI" panel to the dashboard showing the same number the alert uses | ✅ Done (dashboard JSON updated) |
| Add an alert for "a robot-shop deployment has 0 available replicas" (`kube_deployment_status_replicas_available == 0`) | Planned |
| Agree on-call cover and an escalation path, so a firing alert is acted on within minutes | Planned |
| Run a drill after every change to alerting, to check alerts fire for the failures they are meant to catch | Planned |
