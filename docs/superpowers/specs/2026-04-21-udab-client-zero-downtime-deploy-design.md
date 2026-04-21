# udab-client Zero-Downtime Deploy Fix

**Date:** 2026-04-21  
**Status:** Approved  
**Scope:** helm/charts/application/templates/deployment.yaml only

## Problem

udab-client hits 503s during every deploy. Two root causes:

**Cause 1 — Pod termination race (primary)**

Sequence without fix:
1. K8s sends SIGTERM to old pod
2. nginx process exits immediately
3. ALB still routes traffic to that pod for up to `deregistration_delay=10s`
4. nginx is dead → ALB gets connection refused → 503

The `deregistration_delay` is supposed to drain connections before the pod dies, but the pod dies before draining completes.

**Cause 2 — Capacity gap during rollout**

- No explicit rolling update strategy → defaults: `maxUnavailable=1`, `maxSurge=1`
- `successThreshold=2` → new pod needs 2 consecutive probe passes (20s minimum after 10s initial delay = ~30s to Ready)
- During rollout: 1 old pod killed, new pod not Ready for 30s → 3/4 pods serving, 25% capacity loss
- Under load, 3 pods may not be enough → 503s from overload or queue drops

## Solution

Three changes to `deployment.yaml`. No infra changes required.

### Change 1 — preStop lifecycle hook

```yaml
lifecycle:
  preStop:
    exec:
      command: ["sleep", "15"]
```

**Why it works:**

- K8s termination sequence: remove from endpoints → run preStop → send SIGTERM → wait terminationGracePeriodSeconds
- When pod is removed from endpoints, TargetGroupBinding controller deregisters from ALB
- ALB drains existing connections for `deregistration_delay=10s`, then stops routing new traffic
- preStop sleeps 15s > deregistration_delay 10s → nginx is still alive for the full ALB drain window
- SIGTERM arrives at t=15s; ALB already stopped routing at t=10s → no 503s
- terminationGracePeriodSeconds=30s gives 15s of margin after SIGTERM (nginx shuts down in <1s for static files)

**Critical constraint:** preStop sleep (15s) must always be >= deregistration_delay (10s). If deregistration_delay is ever increased, preStop sleep must increase too.

### Change 2 — Rolling update strategy

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxUnavailable: 0
    maxSurge: 1
```

- `maxUnavailable: 0` — never terminate old pod until new pod is Ready; always 4 pods serving
- `maxSurge: 1` — allows 5th pod to start before old one is killed
- Side effect: deploy takes slightly longer (sequential pod replacements with surge pod)

### Change 3 — Readiness probe successThreshold

```yaml
readinessProbe:
  successThreshold: 1  # was: 2
```

- Cuts pod readiness time from ~30s to ~20s (10s initial delay + 1 check × 10s period)
- Less time spent with surge pod occupying a slot before it becomes Ready

## What does NOT change

- `deregistration_delay=10s` — correct as-is; preStop is designed around this value
- ALB health check config — no change needed
- `terminationGracePeriodSeconds: 30` — already sufficient
- `dns.tf`, `stage/app/dns.tf` — no changes

## Files Changed

| File | Change |
|------|--------|
| `helm/charts/application/templates/deployment.yaml` | Add `strategy` block, `lifecycle.preStop`, change `successThreshold: 2→1` |

## Verification

After deploying:
1. Trigger a deploy (push new image tag)
2. Watch ALB target group in AWS console — targets should stay Healthy throughout
3. Run `curl -o /dev/null -s -w "%{http_code}\n"` in a loop during deploy — should see no 503s
4. `kubectl rollout status deployment/udab-client -n prod` — should complete without error
