---
name: willform-build
description: Trigger and monitor builds after code is ready. Use after committing code to Forgejo.
user-invocable: true
---

# Willform Build

After writing code, trigger a build by pushing to Forgejo and monitor until completion.

## When to Use

- Code is written and ready to go live
- User asks to "build", "올려줘", "실행해줘"
- Retrying after a failed build

## Build Trigger Process

### Step 1: Commit and Push (EXACT sequence)

```bash
cd ~/workspace/app

# Protect the build workflow — NEVER modify it
git add -A
git reset HEAD -- .forgejo/ 2>/dev/null || true
git checkout -- .forgejo/ 2>/dev/null || true
git add .forgejo/

git commit -m "feat: description of changes"
git push -u origin main --force-with-lease
```

CRITICAL RULES:
- NEVER create, modify, or delete `.forgejo/workflows/` — the platform manages it
- NEVER try to "fix" builds by editing the workflow file
- If build fails, fix your app code or Dockerfile, not the workflow

### Step 2: Wait for Build

After `git push`, the build starts automatically.
- Typical build time: 2-3 minutes
- Tell user: "준비 중입니다... (보통 2-3분 걸려요)"

### Step 3: Check Build Status

Check Forgejo Actions to see if the build succeeded or failed:
- URL: `{forgejo_internal_url}/{repo_path}/actions`
- Look for the latest workflow run
- Status: success / failure / running

### Step 4: Handle Result

**On success:**
- Tell user: "거의 완성됐습니다! 조금만 기다려주세요."
- Proceed to deployment verification (willform-deploy skill)
- Do NOT say "완료" yet — deployment must finish first

**On failure:**
- Read the build logs to identify the error
- Common failures and fixes below
- Fix the issue, commit, and push again
- Maximum 3 retry attempts before reporting to user

## Common Build Failures

| Symptom | Cause | Fix |
|---------|-------|-----|
| `prisma generate` fails | confbox conflict with Next.js deps | Use isolated prisma-gen stage from DATABASE.md |
| `COPY --from=builder /app/public` fails | public/ directory doesn't exist | Add `mkdir -p public` before `npm run build` in Dockerfile |
| Module not found (dependency) | Missing npm package | Add to package.json, run `npm install` |
| Module not found (@/components/...) | Leftover import from old app | Delete the stale import or remove the file referencing old components. If replacing an app, follow Full App Cleanup in willform-forgejo |
| TypeScript errors | Type errors in code | Fix the type errors in source files |
| `params` does not satisfy `PageProps` | Next.js 15 async params | Change `params: { id: string }` to `params: Promise<{ id: string }>` and add `const { id } = await params` |
| `searchParams` does not satisfy `PageProps` | Next.js 15 async searchParams | Change `searchParams: { q?: string }` to `searchParams: Promise<{ q?: string }>` and add `const { q } = await searchParams` |
| `node:22` fails | confbox/c12 incompatibility | Use `node:20-alpine` (NEVER node:22) |
| OOM during build | Too many deps or large build | Simplify dependencies |

## Dockerfile

DO NOT write your own Dockerfile from scratch. Use the template from DATABASE.md exactly.
The template handles: isolated prisma stage, confbox conflict, standalone output, non-root user.

Key rules:
- `output: "standalone"` must be in next.config.ts
- `mkdir -p public` before `npm run build`
- CMD is `["node", "server.js"]` — NEVER include prisma commands in CMD
- Use `node:20-alpine` only

## Retry Strategy

1. First failure: Read logs, identify root cause, fix, push again
2. Second failure: Try alternative approach (different fix)
3. Third failure: STOP and report to user honestly

After 3 failures, tell user:
- What went wrong (in simple terms, no technical jargon)
- What options are available
- Ask if they want to try a different approach

## FORBIDDEN: Workflow Modification

The `.forgejo/workflows/build.yaml` file is managed by the platform.
Modifying it will break the entire build pipeline.

If you suspect the workflow is broken:
- Do NOT edit it
- Fix your Dockerfile or app code instead
- If the workflow itself is genuinely broken, tell the user to contact support

## User Communication

| Event | Message |
|-------|---------|
| Starting | "준비 중입니다... (보통 2-3분 걸려요)" |
| Success | "거의 완성됐습니다! 조금만 기다려주세요." |
| Failure (retrying) | "문제가 있어서 다시 시도하고 있습니다." |
| Failure (giving up) | "문제가 생겼습니다. [사용자가 이해할 수 있는 설명]" |

FORBIDDEN words in user messages:
Docker, image, container, Harbor, pod, namespace, ArgoCD,
git, commit, push, build, deploy, API, endpoint, schema,
migration, database, server, node, Prisma, pipeline, workflow,
registry, tag, SHA, branch, merge, CI/CD, route, middleware,
Dockerfile, config, standalone, module, dependency, runtime,
Kubernetes, cluster, ingress, manifest, YAML, JSON, CLI,
frontend, backend, fullstack, ORM, query, SQL, seed,
DNS, SSL, certificate

Good: "준비 중입니다", "문제가 있었지만 해결했습니다"
Bad: "Docker 이미지를 빌드하고 Harbor에 푸시 중입니다"

## IMPORTANT

- Build completion is NOT app completion. Deployment must follow.
- NEVER say "완료되었습니다" or provide the app URL at build stage.
- NEVER say "만들었습니다" after code is committed — the app is not live yet.
- After successful build, proceed to deployment verification (willform-deploy).
