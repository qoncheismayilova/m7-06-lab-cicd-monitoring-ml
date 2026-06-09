# Rollback Runbook — Vision Moderation Service

## 🚨 When to Roll Back

Trigger rollback immediately if ANY of the following occur:

- Error rate > 2% for 10 minutes
- p99 latency > 1 second for 10 minutes
- ModelVersionMismatch alert is firing for > 5 minutes
- DriftDetected (PSI > 0.3) AND quality metrics degrade
- Canary deployment shows >5% failure rate vs baseline

---

## 🔧 How to Roll Back

### Option 1 — GitHub Actions rollback

1. Go to GitHub repo
2. Actions tab
3. Select "rollback" workflow
4. Click "Run workflow"
5. Choose previous stable commit SHA
6. Confirm execution

---

### Option 2 — Manual CLI rollback

```bash
bash scripts/rollback.sh production


Sonra:

```md
## What to Verify After Rollback

- Error rate returns to baseline (<0.5%)
- p99 latency < 500ms
- All alerts in Prometheus are resolved
- Canary traffic stable (no spikes)
- Dashboard health = GREEN

## Who to Notify

- On-call engineer (PagerDuty page)
- ML team
- SRE team

## What NOT to Do

- Do NOT deploy a new version immediately after rollback
- Do NOT debug root cause before stabilizing the system
- Do NOT skip canary verification
- Do NOT ignore active alerts

## When to Re-Deploy

- Root cause identified and fixed
- Staging stable for more than 30 minutes
- No active alerts
- Previous failure reproducible in stagingS