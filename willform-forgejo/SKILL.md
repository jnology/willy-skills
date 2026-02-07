---
name: willform-forgejo
description: Git operations for committing and pushing code to the project's Forgejo repository. Use for all code delivery workflows.
user-invocable: false
---

# Willform Forgejo — Code Delivery

How to safely commit and push code to the project's Forgejo repository.

## Workspace Layout

```
~/workspace/app/           # Git working directory (cloned from Forgejo)
├── .forgejo/workflows/    # BUILD WORKFLOW — NEVER touch
├── app/                   # Next.js App Router pages
├── components/            # React components
├── lib/                   # Utilities, actions, db
├── prisma/                # Database schema
├── Dockerfile             # Container build instructions
├── package.json
├── next.config.ts
└── ...
```

## Safe Commit & Push Sequence

ALWAYS follow this exact sequence. Do not skip steps.

```bash
cd ~/workspace/app

# 1. Protect .forgejo/ — restore to committed state
git checkout -- .forgejo/ 2>/dev/null || true

# 2. Stage all changes EXCEPT .forgejo/
git add -A
git reset HEAD -- .forgejo/ 2>/dev/null || true

# 3. Verify .forgejo/ is not in staged changes
git diff --cached --name-only | grep -q "^\.forgejo/" && echo "ABORT: .forgejo modified" && exit 1

# 4. Commit with descriptive message
git commit -m "feat: add product listing and cart"

# 5. Push (force-with-lease to handle diverged history safely)
git push -u origin main --force-with-lease
```

### Why `--force-with-lease`?

The workspace may have diverged from remote (e.g., platform updated workflow files). `--force-with-lease` safely overwrites only if no unexpected remote changes exist. If it fails, pull first then retry.

## Full App Cleanup (Before Replacement)

When replacing an entire app (e.g., shopping mall → blog), clean the workspace BEFORE writing new code:

```bash
cd ~/workspace/app

# 1. Delete old app directories
rm -rf app/
rm -rf components/
rm -f lib/actions.ts
rm -f lib/utils.ts
rm -f prisma/schema.prisma

# 2. NEVER delete these (platform-managed or reusable)
# .forgejo/     — build workflow
# lib/db.ts     — Prisma singleton (reusable)
# Dockerfile    — will be rewritten
# package.json  — will be rewritten
# next.config.ts, tsconfig.json, tailwind.config.ts — usually unchanged
# .gitignore, .dockerignore

# 3. Stage deletions
git add -A
# Don't commit yet — write new code first, then commit everything together
```

**IMPORTANT**: Deletions and new code must be in the SAME commit. Never commit just deletions — that would leave the repo in a broken state.

## Pre-Commit Checklist

Before committing, verify ALL of these:

1. **All referenced files exist** — every import/require points to a real file
2. **package.json includes all dependencies** — run `npm install` if you added new packages
3. **prisma/schema.prisma is valid** — run `npx prisma generate` if schema changed
4. **Dockerfile exists** — copy from DATABASE.md template, never write from scratch
5. **next.config.ts has `output: "standalone"`**
6. **.forgejo/workflows/ is untouched** — verify with `git diff --name-only | grep .forgejo` (should be empty)

## Error Recovery

### Push Rejected (non-fast-forward)

```bash
# Pull remote changes, keep our work on top
git pull --rebase origin main

# If rebase conflicts: accept our version (we own the code)
git checkout --theirs .
git add -A
git rebase --continue

# Retry push
git push -u origin main --force-with-lease
```

### Push Rejected (force-with-lease failed)

Remote was updated between your last fetch and push. This is rare.

```bash
# Fetch latest state
git fetch origin

# Rebase on top of remote
git rebase origin/main

# Push again
git push -u origin main --force-with-lease
```

### Accidentally Modified .forgejo/

```bash
# Restore workflow files to their committed state
git checkout HEAD -- .forgejo/

# If .forgejo/ was deleted or never existed, do NOT recreate it
# The platform manages these files — they will be restored on next platform sync
```

### Commit Has No Changes

```bash
# Check what's happening
git status

# If working directory is clean, there's nothing to commit — this is fine
# If files exist but are untracked, make sure you ran `git add -A`
```

### Large File Errors

Forgejo has a file size limit. If push fails due to large files:

```bash
# Check for large files
find . -type f -size +50M -not -path './.git/*'

# Remove from tracking (not from disk)
git rm --cached path/to/large-file
git commit -m "fix: remove large file from tracking"
```

## Rules

### NEVER Do
- Create, modify, or delete `.forgejo/workflows/` files
- Run `git init` — the repo is already initialized
- Push to branches other than `main` (unless explicitly creating a feature branch)
- Include `node_modules/`, `.next/`, or `.env` in commits
- Commit partial work — all files for a feature must be committed together

### ALWAYS Do
- Run the safe commit sequence above (protect .forgejo, stage, verify, commit, push)
- Write descriptive commit messages in English (`feat:`, `fix:`, `refactor:`)
- Commit ALL files needed for a complete feature in a single commit
- Verify no missing file references before committing
- Use `--force-with-lease` (never bare `--force`)

## Commit Message Format

```
feat: add shopping cart with quantity controls
fix: resolve product image display issue
refactor: simplify checkout flow
```

Keep messages concise and descriptive. Use conventional commit prefixes.

## Approval Flow Integration

After committing, behavior depends on the project's approval level:

| Level | After Commit |
|-------|-------------|
| `auto` | Push immediately, build starts automatically |
| `notify` | Push immediately, notify user that changes are live |
| `confirm` | Describe changes in simple terms, wait for user approval, then push |
| `manual` | Describe changes in detail, wait for explicit approval, then push |

When describing changes for approval:
- Use feature descriptions, not file lists
- Example: "상품 목록과 장바구니 기능을 추가했습니다. 확인해주시겠어요?"
- NEVER show code diffs, file paths, or technical details to the user
