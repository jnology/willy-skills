---
name: willform-guard
description: Security rules for Willform - enforces namespace isolation and platform policies
user-invocable: false
disable-model-invocation: false
---

# Willform Security Guard

You are operating within a **Willform project namespace**. Follow these security rules strictly.

## Namespace Isolation (CRITICAL)

**You MUST NOT access resources outside this namespace.**

### Blocked Actions
- Accessing other namespaces' pods, services, or data
- Connecting to cluster-wide resources
- Reading other projects' Forgejo repositories
- Accessing Kubernetes nodes or host network

### Allowed Actions
- All resources within THIS namespace (your project)
- Central Forgejo instance (willform-system namespace, your org only)
- Same-namespace database
- External AI APIs (openrouter.ai)
- Willform platform API

## User Space Freedom

Within this namespace, the user has full control:
- Read/write any files in the workspace
- Access environment variables and secrets (their own)
- Install packages and dependencies
- Run any code (within resource limits)

**Do NOT restrict file access within the workspace** - this is the user's space.

## Resource Awareness

This namespace has resource limits applied:
- CPU: Limited by ResourceQuota
- Memory: Limited by ResourceQuota
- Storage: Limited by PVC size

Warn the user if operations might exceed limits.

## Code Generation Guidelines

When generating code:
1. Do not include hardcoded credentials
2. Use environment variables for sensitive data
3. Respect the project's existing code style
4. Follow the approval level set by the user

## Approval Workflow

Respect the user's `approvalLevel` setting:
- `auto`: Execute immediately
- `notify`: Execute and notify
- `confirm`: Ask before executing (default)
- `manual`: Require detailed review
