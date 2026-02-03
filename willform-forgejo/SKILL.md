---
name: willform-forgejo
description: Git operations with project's Forgejo instance - commit, branch, PR management
user-invocable: true
---

# Willform Forgejo Integration

Manage code in the project's Forgejo Git repository.

## Available Commands

When the user asks about code management, use these capabilities:

### File Operations
- **Read file**: Fetch file content from repository
- **Create/Update file**: Create new files or modify existing ones
- **Delete file**: Remove files from repository

### Git Operations
- **Commit**: Save changes with a descriptive message
- **Branch**: Create, list, or switch branches
- **Merge**: Merge branches (requires approval based on level)

### Pull Request
- **Create PR**: Open a pull request for review
- **List PRs**: Show open pull requests
- **Merge PR**: Merge an approved pull request

## Workflow

1. User describes what they want to build
2. You generate the code
3. Based on approval level:
   - `auto`: Commit directly
   - `notify`: Commit and notify user
   - `confirm`: Show diff and ask for approval
   - `manual`: Show detailed review before commit

## Best Practices

When committing code:
- Write clear, descriptive commit messages
- Group related changes in single commits
- Create feature branches for major changes
- Keep commits atomic and focused

## Example Interactions

User: "Create a login page"
→ Generate code → Show preview → Get approval → Commit to Forgejo

User: "Show me the current code"
→ Fetch from Forgejo → Display to user

User: "Create a branch for the new feature"
→ Create branch in Forgejo → Switch context
