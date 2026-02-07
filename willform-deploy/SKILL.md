---
name: willform-deploy
description: Verify deployment health after build completes. Use to check if the app is live and accessible.
user-invocable: true
---

# Willform Deploy

After a successful build, verify that the app is live and accessible.

## When to Use

- Build just completed successfully
- User asks "내 앱 상태", "확인해줘", "안 되는데"
- Checking if app is running after deployment
- Troubleshooting a non-responsive app

## How Deployment Works

After a successful build, the platform automatically updates the app.
You do NOT need to trigger deployment manually — it happens automatically.

Your job: verify that the app is healthy and tell the user when it's ready.

## Deployment Verification Process

### Step 1: Check App Status

```bash
/data/bin/kubectl get pods -n {userNamespace} -l app={projectId}
```

Expected: pods in `Running` status with `1/1` ready and `0` restarts.

Wait up to 3 minutes for pods to become ready. Check every 15 seconds.

### Step 2: Check for Problems

If pods are NOT Running, check the reason:

```bash
# Get pod details
/data/bin/kubectl describe pod -n {userNamespace} -l app={projectId}

# Get app logs
/data/bin/kubectl logs -n {userNamespace} -l app={projectId} --tail=30
```

### Step 3: Verify App Responds

Once pods are Running, confirm the app is accessible:

```bash
curl -sf https://{appDomain} --max-time 10
```

Must return HTTP 200. If not, the app may still be starting — retry up to 3 times with 10-second intervals.

### Step 4: Report to User

ONLY after ALL checks pass (pods Running + HTTP 200):
- "완료되었습니다! https://{appDomain} 에서 확인하실 수 있습니다."

## Troubleshooting

### CrashLoopBackOff (App keeps restarting)

1. Read logs: `/data/bin/kubectl logs -n {userNamespace} -l app={projectId} --tail=50`
2. Common causes:
   - Missing DATABASE_URL → check if `platform-secrets` exists in namespace
   - Prisma client not generated → Dockerfile missing prisma-gen stage (see DATABASE.md)
   - Port mismatch → app must listen on port 3000
   - Module not found → missing dependency in package.json
3. Fix the code/Dockerfile, commit, push again
4. Tell user: "문제가 있어서 수정하고 있습니다."

### ImagePullBackOff (Build result not found)

1. The build may not have completed successfully
2. Check Forgejo Actions for build status
3. If build failed, fix and rebuild (see willform-build skill)
4. Tell user: "준비 과정에서 문제가 생겨서 다시 시도하고 있습니다."

### Pending (App stuck waiting)

1. Usually resolves within 1-2 minutes
2. If stuck for >3 minutes, check events:
   ```bash
   /data/bin/kubectl get events -n {userNamespace} --sort-by='.lastTimestamp' | tail -10
   ```
3. Common causes: resource limits, node capacity

### App Returns Non-200

1. Check logs for runtime errors
2. Common causes:
   - Database connection failed (DATABASE_URL wrong or DB not ready)
   - Missing environment variables
   - Build succeeded but app has runtime bugs
3. Fix, commit, push, wait for redeploy

## Retry Strategy

1. First failure: Diagnose from logs, fix, rebuild
2. Second failure: Try different approach
3. Third failure: STOP and report to user

Tell user honestly what happened (in simple terms):
- "앱 시작 과정에서 문제가 생겼습니다. 다른 방법을 시도해볼까요?"
- "데이터 연결에 문제가 있습니다. 확인 후 다시 시도하겠습니다."

## Verification Checklist

Before saying "완료되었습니다":
- [ ] Pods are `Running` with `1/1` Ready
- [ ] Pod restart count is `0`
- [ ] `curl -sf https://{appDomain}` returns HTTP 200
- [ ] ALL three checks passed

If ANY check fails, do NOT report completion. Diagnose and fix first.

## User Communication

Output each message as plain text BEFORE the tool calls for that phase.
Text between tool calls is delivered to the user immediately.

| Event | Output text | Then do |
|-------|-------------|---------|
| Checking status | "마지막 준비 중입니다..." | kubectl get pods, curl health check |
| All healthy | "완료되었습니다! https://{appDomain} 에서 확인하실 수 있습니다." | (final) |
| Problem found, fixing | "문제가 있어서 수정하고 있습니다." | Fix and rebuild |
| Problem found, need help | "문제가 생겼습니다. [사용자가 이해할 수 있는 설명]" | (wait for user) |
| App down (user reported) | "확인해보겠습니다." | Diagnose, fix or report |

FORBIDDEN words: See willform-guard SKILL.

Good: "완료되었습니다!", "문제가 있어서 수정하고 있습니다", "확인해보겠습니다"
Bad: "Pod가 CrashLoopBackOff 상태입니다", "ArgoCD 동기화를 기다리고 있습니다"

## IMPORTANT

- NEVER say "완료" or provide URL before all verification checks pass.
- NEVER use future tense with URL ("배포가 완료되면 ~에서 확인" is FORBIDDEN).
- URL is given ONLY after confirming the app is live and responding.
- If the app was already deployed and user asks to update, the same process applies: verify after build.
- When auto-recovering from errors, tell user "문제가 있었지만 해결했습니다" — no technical details.
