# Rollback Runbook — Vision Moderation

> **Target audience:** on-call engineer, 3 a.m., under pressure.
> Complete this in < 10 minutes.

---

## 1 · When to roll back

Trigger a rollback immediately if **any** of the following fire in production:

| Signal | Threshold |
|---|---|
| `AvailabilityBurnFast` alert | Paging (burn rate > 14.4×) |
| `LatencyP99High` alert | p99 > 1 s for ≥ 5 min |
| `ModelVersionMismatch` alert | Any replica, ≥ 5 min |
| Manual error rate check | > 1 % over a 10-min window |
| Canary verify script | Non-zero exit after deploy |

---

## 2 · How to roll back

### Option A — GitHub Actions (preferred)

1. Go to **Actions → Deploy Vision Moderation Model → Run workflow**
2. Set `ref` = last known-good SHA (check Slack `#deploys` or the GHCR tag list)
3. Set `ROLLBACK=true` environment variable (workflow skips canary, goes straight to full deploy)
4. Click **Run workflow** — requires production environment approver

### Option B — CLI (if Actions is unavailable)

```bash
# 1. Set the target SHA to the last known-good image
export GOOD_SHA=<previous-sha>
export IMAGE_TAG=$GOOD_SHA
export DEPLOY_ENV=production
export CANARY_WEIGHT=100

# 2. Execute rollback script
bash ./scripts/rollback.sh

# 3. Confirm pods are rolling
kubectl rollout status deployment/vision-moderation -n production --timeout=120s
```

---

## 3 · What to verify (must all return to green)

- [ ] `AvailabilityBurnFast` — resolved in Alertmanager
- [ ] `LatencyP99High` — resolved in Alertmanager
- [ ] `ModelVersionMismatch` — resolved in Alertmanager
- [ ] Grafana SLO dashboard: error rate < 0.5 % over 5-min rolling window
- [ ] Grafana latency dashboard: p99 < 1 s
- [ ] All production pods report the expected `model_version` header

---

## 4 · Who to notify

| When | Channel | Who |
|---|---|---|
| Rollback initiated | `#incidents` Slack | Post: SHA being rolled back, SHA being deployed |
| Rollback complete | `#incidents` Slack | Post: verification checklist results |
| SLO breach confirmed | PagerDuty | Escalate to ML Platform lead |
| Customer impact likely | `#customer-success` Slack | ML Platform lead notifies CS team |

---

## 5 · What NOT to do

- ❌ **Do not roll forward** before root cause is identified — a second bad deploy compounds the incident
- ❌ **Do not skip canary** unless the rollback script itself (`rollback.sh`) fails — even rollbacks benefit from a quick sanity check
- ❌ **Do not modify `rollback.sh`** inline during the incident — use the version in the repo
- ❌ **Do not close the PagerDuty incident** until all alerts are green and a post-mortem is scheduled

---

## 6 · When to roll forward

Re-deploy only when **all** of the following are true:

- [ ] Root cause is identified and documented in the incident ticket
- [ ] Fix is merged to `main` and CI is green (including Trivy scan)
- [ ] At least one team member (not the author) has reviewed the fix
- [ ] Staging smoke tests pass on the fixed image
- [ ] On-call lead has approved the forward deploy in the `#incidents` thread