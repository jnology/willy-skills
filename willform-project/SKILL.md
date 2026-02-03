---
name: willform-project
description: Project management - build, deploy, logs, and status monitoring
user-invocable: true
---

# Willform Project Management

Manage the project's build, deployment, and monitoring.

## Available Commands

### Build
- **Trigger build**: Start a new build from current code
- **Build status**: Check ongoing or recent build status
- **Build logs**: View build output and errors

### Deployment
- **Deploy**: Deploy the latest successful build
- **Rollback**: Revert to a previous version
- **Deploy status**: Check current deployment state

### Monitoring
- **Status**: Overview of project health
- **Logs**: View application runtime logs
- **Resources**: Check CPU/memory usage

### Environment
- **Set env var**: Configure environment variables
- **List env vars**: Show current configuration
- **Secrets**: Manage sensitive configuration (masked)

## Workflow

### When user wants to deploy:
1. Check if there are uncommitted changes
2. Trigger build if needed
3. Wait for build completion
4. Deploy to the project's namespace
5. Report deployment status

### When user asks about status:
1. Check pod status in namespace
2. Check ArgoCD sync status
3. Report any issues or errors

## Status Indicators

- ✅ **Healthy**: All systems operational
- 🔄 **Building**: Build in progress
- 🚀 **Deploying**: Deployment in progress
- ⚠️ **Warning**: Minor issues detected
- ❌ **Error**: Action required

## Example Interactions

User: "Deploy my app"
→ Check code → Build → Deploy → Report status

User: "Why is my app not working?"
→ Check pod status → Check logs → Diagnose → Suggest fix

User: "Show me the logs"
→ Fetch recent logs → Display with formatting
