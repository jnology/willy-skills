---
name: willform-build
description: Trigger and monitor builds after code is ready. Use after committing code to Forgejo.
user-invocable: true
---

# Willform Build

After writing code, trigger a build by pushing to Forgejo and monitor until completion.

## When to Use

- Code is written and ready to go live
- User asks to "build", "make it live", "run it"
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
- Output text: "Getting things ready... (usually takes 2-3 minutes)" BEFORE checking build status

### Step 3: Check Build Status

Check Forgejo Actions to see if the build succeeded or failed:
- URL: `{forgejo_internal_url}/{repo_path}/actions`
- Look for the latest workflow run
- Status: success / failure / running

### Step 4: Handle Result

**On success:**
- Tell user: "Almost done! Just a moment."
- Proceed to deployment verification (willform-deploy skill)
- Do NOT say "done" yet — deployment must finish first

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
| `Can't reach database server` at build | `prisma db push` in package.json build script | Remove `prisma db push` from build script. Use `"build": "next build"` only. DB migrations are handled by CI |
| `node:22` fails | confbox/c12 incompatibility | Use `node:20-alpine` (NEVER node:22) |
| `"build": "prisma db push && next build"` | prisma db push needs DB at build time | Use `"build": "next build"` only. prisma generate is in Dockerfile |
| Missing `tailwind-merge` or `clsx` | `cn()` utility requires these packages | Add `tailwind-merge` and `clsx` to package.json dependencies |
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
| Starting | "Getting things ready... (usually takes 2-3 minutes)" |
| Success | "Almost done! Just a moment." |
| Failure (retrying) | "Ran into an issue. Trying again now." |
| Failure (giving up) | "Something went wrong. [simple explanation the user can understand]" |

FORBIDDEN words: See willform-guard SKILL.

Good: "Getting things ready", "Ran into a small issue, but it's fixed now"
Bad: "Building Docker image and pushing to Harbor"

## IMPORTANT

- Build completion is NOT app completion. Deployment must follow.
- NEVER say "All done" or provide the app URL at build stage.
- NEVER say "I've built it" after code is committed — the app is not live yet.
- After successful build, proceed to deployment verification (willform-deploy).
