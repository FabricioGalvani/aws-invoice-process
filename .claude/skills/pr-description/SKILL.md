---
name: pr-description
description: Writes pull request descriptions following Conventional Commits standard. Use when creating a PR, writing a PR, or when the user asks to summarize changes for a pull request.
---

When writing a PR description:

1. Run `git diff main...HEAD` to see all changes on this branch
2. Run `git log main...HEAD --oneline` to review commit history for context
3. Determine the appropriate Conventional Commit type based on the changes (see below)
4. Write the PR title and description following the format below

## PR Title Format

The PR title MUST follow Conventional Commits:
Examples:
- `feat(auth): add OAuth2 login support`
- `fix(api): handle null response from user endpoint`
- `refactor(db): extract query builder into separate module`
- `docs: update installation instructions`
- `chore(deps): bump axios to 1.7.0`

### Conventional Commit Types

- **feat**: A new feature for the user
- **fix**: A bug fix
- **refactor**: Code change that neither fixes a bug nor adds a feature
- **perf**: A code change that improves performance
- **docs**: Documentation only changes
- **style**: Changes that don't affect code meaning (formatting, whitespace, etc.)
- **test**: Adding or correcting tests
- **build**: Changes to build system or external dependencies
- **ci**: Changes to CI configuration files and scripts
- **chore**: Other changes that don't modify src or test files
- **revert**: Reverts a previous commit

### Breaking Changes

If the PR introduces breaking changes, append `!` after the type/scope:
- `feat(api)!: change response format for /users endpoint`

And include a `BREAKING CHANGE:` section in the description explaining the impact and migration path.

## PR Description Format

## What
One sentence explaining what this PR does.

## Why
Brief context on why this change is needed.

## Changes
- Bullet points of specific changes made
- Group related changes together
- Mention any files deleted or renamed

## Breaking Changes (only if applicable)
Describe what breaks and how users should migrate.