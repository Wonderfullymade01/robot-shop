# Runbook: RobotShopAvailabilitySLOBurn

**Alert:** `RobotShopAvailabilitySLOBurn` (severity: critical, posted to Slack `#alerts`)
**Meaning:** more than **14.4%** of requests to web, catalogue, cart or payment have failed (HTTP 5xx) for at least
2 minutes. At this rate the 30-day error budget (1% of requests) would be used up in about 2 days.
See the [SLO and error budget policy](slo-error-budget-policy.md).

## 1. Acknowledge (first 2 minutes)
- Reply in `#alerts`: "Looking at it" so others know someone is on it.
- Note the time the alert fired. You will need it for the postmortem.

## 2. Check the impact
Open Grafana:
```bash
kubectl -n monitoring port-forward svc/monitoring-grafana 3000:80
```
http://localhost:3000 → dashboard **Robot Shop RED**:
- **Error rate (%)**: which service is failing?
- **Request rate**: did traffic drop to zero for a service (it is down) or is it failing while still busy?
- **Latency p95**: is something slow rather than failing?

Check what Kubernetes sees:
```bash
kubectl get pods,deploy -n robot-shop
helm history robot-shop -n robot-shop
```

## 3. Mitigate first: get users working again before finding the cause
| What you see | Do this |
|---|---|
| A deploy happened just before the alert | `helm rollback robot-shop <last good revision> -n robot-shop --wait` |
| Pods in `CrashLoopBackOff` / `ImagePullBackOff` | Roll back as above, or `kubectl rollout undo deploy/<service> -n robot-shop` |
| A deployment has 0 ready pods | `kubectl scale deploy/<service> -n robot-shop --replicas=1` |
| Pods running but stuck or slow | `kubectl rollout restart deploy/<service> -n robot-shop` |
| `OOMKilled` restarts | Raise the memory limit in the chart and `helm upgrade` |

Confirm on the dashboard that errors are falling. Slack posts **[RESOLVED]** a few minutes after the error rate recovers.

## 4. Find the root cause
- **Logs (Loki)**, in Grafana → Explore → Loki:
  - `{namespace="robot-shop", app="<service>", level="error"}`
  - `{namespace="robot-shop", app="web"} |= "upstream"` (nginx cannot reach a backend)
- **Traces (Jaeger)**: `kubectl -n monitoring port-forward svc/jaeger 16686:16686` → http://localhost:16686 →
  service `payment` or `traefik` → find a slow or failing request and see which hop is to blame.
- **Kubernetes**: `kubectl describe pod <pod> -n robot-shop` (events, restarts, OOM), `kubectl logs <pod> -n robot-shop`.
- **Network**: test a backend from inside the cluster:
  `kubectl run curltest -n robot-shop --rm -i --restart=Never --image=curlimages/curl -- curl -m 5 http://<service>:<port>/`

## 5. Communicate
Post short updates in `#alerts`: what is broken, what you did, current status. Say when it is resolved.

## 6. Afterwards
Write a **blameless postmortem** in `docs/` (timeline, impact, root cause, what we will change), even for small incidents.
