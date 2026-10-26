# Robot Shop: Availability SLO and Error Budget Policy

## Service level indicator (SLI)
**Availability** = successful requests ÷ all requests, measured at the Traefik front door for the four user-facing
services: **web, catalogue, cart and payment**. A request is *unsuccessful* if it returns an HTTP **5xx** status
(a server-side failure). 4xx responses are client errors and do not count against the SLO.

Source metric: `traefik_service_requests_total` in Prometheus
(recording rule `robotshop:error_ratio:rate5m` in `observability/slo-rules.yaml`).

## Service level objective (SLO)
**99% of requests succeed, measured over a rolling 30 days.**

## Error budget
The budget is the 1% of requests that are allowed to fail. If all failures happened at once, that would be about
**7.2 hours of complete outage per 30 days** (1% × 30 days × 24 hours).

## Alerting
`RobotShopAvailabilitySLOBurn` fires when the 5-minute error ratio stays above **14.4%** for 2 minutes. That is a burn
rate of 14.4×: at that speed the whole 30-day budget would be gone in about 2 days. The alert goes to Slack `#alerts`
and links to the [runbook](runbook.md).

## Policy: what the team does as the budget is used
| Budget remaining (30 days) | Releases | Engineering focus |
|---|---|---|
| **More than 50%** | Normal releases | Features as planned |
| **25% to 50%** | Releases continue with extra review; risky changes are postponed | Fix the top cause of recent errors |
| **Less than 25%** | Only bug fixes and reliability work are released | Reliability work takes priority over features |
| **0% (budget spent)** | **Release freeze** until the budget recovers above 25% | Postmortem for every incident in the window; agree fixes before unfreezing |

## Exceptions
- **Security fixes** may always be released.
- Failures caused by a dependency outside our control (for example the external payment gateway) are still counted,
  but are reviewed separately when deciding on a freeze.

## Ownership and review
- **Owner:** the Robot Shop team (for this capstone: Fayomi Fiyinfoluwa).
- The SLO, the alert threshold and this policy are reviewed **once a month**, and after every major incident.
