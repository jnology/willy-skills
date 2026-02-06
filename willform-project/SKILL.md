---
name: willform-project
description: Project management - build, deploy, logs, and status monitoring
user-invocable: true
---

# Willform Project Management

Manage the project's build, deployment, and monitoring.

## Build Pipeline

Builds are triggered automatically by **Forgejo Actions** when code is pushed.
- Image: `harbor-dev.willform.app/willform/{projectId}:{sha}`
- Workflow: `.forgejo/workflows/build.yaml` (auto-generated)

## Available Commands

### Build
- **Build status**: Check ongoing or recent build status via Forgejo Actions
- **Build logs**: View build output and errors

### Deployment
- **Deploy**: Push code to trigger auto-build and deploy
- **Rollback**: Revert to a previous version via ArgoCD
- **Deploy status**: Check current deployment state

### Monitoring
- **Status**: Overview of project health (pod status, ArgoCD sync)
- **Logs**: View application runtime logs
- **Resources**: Check CPU/memory usage against quota

### Environment
- **Set env var**: Configure environment variables
- **List env vars**: Show current configuration
- **Secrets**: Manage sensitive configuration (masked)

## Workflow

### When user wants to deploy:
1. Check if there are uncommitted changes
2. Commit and push to Forgejo (triggers Forgejo Actions build)
3. Wait for build completion
4. ArgoCD auto-syncs to deploy
5. Report deployment status

### When user asks about status:
1. Check pod status in user namespace
2. Check ArgoCD sync status
3. Report any issues or errors
