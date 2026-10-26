# Postmortem: web requests failing with 504 (ratings unreachable across nodes)

**Date:** 22–24 October 2026 · **Status:** resolved · **Severity:** high · **Format:** blameless

## Summary
About **14% of all requests** through the shop's front door failed with **HTTP 504 Gateway Timeout**, and web's p95
latency was stuck at **5 seconds**. Product ratings could not be loaded or saved. The cause was not the ratings
service itself: the EKS node security group only allowed node-to-node traffic on ports **1025–65535**, and ratings
listens on port **80**. Whenever web and ratings ran on different nodes, the connection was silently dropped.
Fixed by allowing all traffic between nodes in Terraform.

## Impact
- **Affected:** product ratings (fetch and rate) via `/api/ratings/...` on web. Users saw slow pages and missing ratings.
- **Not affected:** catalogue, cart, user and payment calls (they listen on ports above 1024).
- **Size:** error ratio at the front door about **14%**, all 504s. For comparison, the SLO allows 1%.

## Timeline (times approximate)
| When | What happened |
|---|---|
| 22 Oct, ~16:30 | Robot Shop deployed on EKS behind Traefik; load generator started. |
| 22 Oct, ~17:15 | The new RED dashboard shows web errors up to 90% and p95 latency flat at 5 s. |
| 22 Oct, ~18:50 | Logs show 504s on `/api/ratings/...`. **First hypothesis: ratings is starved of CPU** (limit 0.2 CPU). Its limits were raised. No improvement. |
| 22 Oct, ~19:00 | Two unrelated issues found and fixed on the way: the load generator was `OOMKilled` (19 restarts), and the load generator's deliberate error mode (`ERROR=1`) was inflating payment errors. |
| 24 Oct, ~21:45 | After a rebuild, the baseline error ratio measured at **14.35%**, so the problem was still there. |
| 24 Oct, ~21:50 | nginx log on web: `upstream timed out (110: Connection timed out) while connecting to upstream`, i.e. it could not even open a connection. A test pod could not reach `ratings:80` (20 s timeout), although ratings was Ready and its Service and ports were correct. |
| 24 Oct, ~21:55 | **Root cause identified:** node security group rule `ingress_nodes_ephemeral` only allows ports 1025–65535 between nodes. |
| 24 Oct, ~22:00 | Terraform rule `ingress_self_all` added (all ports, only from nodes in the same group). `ratings:80` answered in 38 ms. Error ratio fell to ~0%. |

## Root cause
The EKS Terraform module (`terraform-aws-modules/eks`) creates a node security group that, by default, allows
node-to-node traffic only on ephemeral ports **1025–65535**. On EKS with the VPC CNI, pods use real VPC IP
addresses, so pod-to-pod traffic between nodes is filtered by this security group. Robot Shop's ratings service
(PHP/Apache) listens on port **80**, so traffic to it from pods on *other* nodes was dropped.

## Why it was hard to see
- **The readiness probe still passed.** It is run by the kubelet on the *same* node, where the security group does
  not apply, so Kubernetes reported ratings as healthy.
- **It was intermittent.** It only failed when web and ratings were scheduled on different nodes.
- **Other problems were mixed in.** Load-generator crashes and deliberate payment errors made the error graph noisy,
  and the first fix (more CPU) looked reasonable.

## What went well
- The RED dashboard showed the problem immediately and pointed at web.
- Logs in Loki and nginx's own error message ("while connecting to upstream") separated a network failure from
  a slow application.

## What we will change
| Action | Status |
|---|---|
| Allow all node-to-node traffic in the node security group (`node_security_group_additional_rules`) | ✅ Done |
| Add the availability SLO alert, so a 14% error rate pages someone instead of waiting to be noticed | ✅ Done (`RobotShopAvailabilitySLOBurn`) |
| Test service-to-service connectivity from *another* node after each cluster build (a smoke test via the Service, not only readiness probes) | Planned |
| Note the module's default security-group behaviour in the Terraform README | Planned |
| When an error rate jumps, check the error message ("connect" vs "read" timeout) before tuning resources | Learned |
