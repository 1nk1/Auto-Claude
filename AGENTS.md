# AGENTS.md

Quick reference guide for AI coding agents working in the Auto Claude repository.

## Build, Lint, and Test Commands

### Python Backend (apps/backend/)

```bash
# Run all tests
npm run test:backend
# Or: apps/backend/.venv/bin/pytest tests/ -v

# Run single test file
apps/backend/.venv/bin/pytest tests/test_security.py -v

# Run specific test
apps/backend/.venv/bin/pytest tests/test_security.py::test_bash_command_validation -v

# Skip slow tests
apps/backend/.venv/bin/pytest tests/ -m "not slow"

# Lint (auto-fix)
cd apps/backend && ruff check --fix .

# Format
cd apps/backend && ruff format .
```

### Frontend (apps/frontend/)

```bash
cd apps/frontend

# Run unit tests
npm test

# Run tests in watch mode
npm run test:watch

# Lint
npm run lint

# Type check
npm run typecheck

# Fix linting issues
npm run lint:fix
```

### Pre-commit Hooks

```bash
# Run all checks on all files
pre-commit run --all-files

# Run specific hook
pre-commit run ruff --all-files
```

## Code Style Guidelines

### Python (apps/backend/)

**Imports:**
```python
# Standard library first
import logging
import os
from pathlib import Path
from typing import Any

# Third-party next
from claude_agent_sdk import ClaudeSDKClient

# Local imports last
from core.client import create_client
from core.security import SecurityManager
```

**Type Hints:**
```python
def get_next_chunk(spec_dir: Path) -> dict | None:
    """
    Find the next pending chunk in the implementation plan.

    Args:
        spec_dir: Path to the spec directory

    Returns:
        The next chunk dict or None if all chunks are complete
    """
    ...
```

**Naming Conventions:**
- Functions/variables: `snake_case`
- Classes: `PascalCase`
- Constants: `UPPER_SNAKE_CASE`
- Private: prefix with `_`

**Error Handling:**
```python
# Use specific exceptions
try:
    result = risky_operation()
except FileNotFoundError:
    logger.error("File not found: %s", file_path)
    raise
except Exception as e:
    logger.exception("Unexpected error")
    raise RuntimeError("Operation failed") from e
```

**Formatting:**
- 4 spaces for indentation
- Double quotes for strings
- Line length: flexible (handled by formatter)
- Docstrings for public functions/classes

### TypeScript/React (apps/frontend/)

**Imports:**
```typescript
// React hooks first
import { useState, useEffect, useCallback, memo } from 'react';

// Third-party libraries
import { useTranslation } from 'react-i18next';

// UI components
import { Card, CardContent } from './ui/card';
import { Button } from './ui/button';

// Local utilities/types
import { cn, formatRelativeTime } from '../lib/utils';
import type { Task, TaskStatus } from '../../shared/types';
```

**Component Structure:**
```typescript
// Use named exports (NOT default exports)
export function TaskCard({ task, onClick }: TaskCardProps) {
  const { t } = useTranslation('tasks'); // i18n required for all UI text
  const [isLoading, setIsLoading] = useState(false);

  // Use useCallback for event handlers
  const handleClick = useCallback(() => {
    onClick(task.id);
  }, [task.id, onClick]);

  return (
    <Card>
      {/* Always use translation keys, NEVER hardcoded strings */}
      <span>{t('tasks:status.completed')}</span>
    </Card>
  );
}
```

**Internationalization (i18n):**
- **CRITICAL**: All user-facing text MUST use translation keys
- Translation files: `apps/frontend/src/shared/i18n/locales/{en,fr}/*.json`
- Pattern: `t('namespace:section.key')`
- Example: `t('navigation:items.githubPRs')` ✅ NOT `"GitHub PRs"` ❌

**Naming Conventions:**
- Components: `PascalCase`
- Functions/variables: `camelCase`
- Constants: `UPPER_SNAKE_CASE`
- Types/interfaces: `PascalCase`

**Type Safety:**
```typescript
// Always define prop types
interface TaskCardProps {
  task: Task;
  onClick: () => void;
}

// Use strict TypeScript - avoid 'any'
// Warn on @typescript-eslint/no-explicit-any
```

**Formatting:**
- 2 spaces for indentation
- Single quotes for strings (auto-formatted)
- Prefer `const` over `let`

## Claude Agent SDK Usage

**CRITICAL: Always use `create_client()` from `core.client`, NEVER use the Anthropic API directly.**

```python
from core.client import create_client

# Create SDK client (NOT raw Anthropic API client)
client = create_client(
    project_dir=project_dir,
    spec_dir=spec_dir,
    model="claude-sonnet-4-5-20250929",
    agent_type="coder",  # or "planner", "qa_reviewer", "qa_fixer"
    max_thinking_tokens=None
)

# Run agent session
response = client.create_agent_session(
    name="coder-agent-session",
    starting_message="Implement the authentication feature"
)
```

**Why use the SDK:**
- Pre-configured security (sandbox, allowlists, hooks)
- Automatic MCP server integration
- Tool permissions based on agent role
- Session management and recovery

## Git Workflow

**Branch Targets:**
- **ALL PRs target `develop`** (NOT `main`)
- Only maintainers merge to `main` via `release/*` or `hotfix/*`

**Branch Naming:**
```bash
feature/add-dark-mode       # New features
fix/memory-leak-in-worker   # Bug fixes
docs/update-readme          # Documentation
refactor/simplify-auth      # Code refactoring
test/add-integration-tests  # Test additions
```

**Commit Messages:**
```
<type>: <subject>

<body>

<footer>
```

Examples:
```bash
feat: Add retry logic for failed API calls

Implements exponential backoff for transient failures.
Fixes #123

# Types: feat, fix, docs, style, refactor, test, chore
```

## Common Patterns

### Memory System (Graphiti)

```python
from integrations.graphiti.memory import get_graphiti_memory

memory = get_graphiti_memory(spec_dir, project_dir)
context = memory.get_context_for_session("Implementing feature X")
memory.add_session_insight("Pattern: use React hooks for state")
```

### Security Model

Three-layer defense:
1. OS Sandbox - Bash command isolation
2. Filesystem Permissions - Project directory only
3. Command Allowlist - Dynamic from project analysis

```python
from core.security import SecurityManager

security = SecurityManager(project_dir)
security.validate_bash_command("ls -la")  # Checks allowlist
```

### E2E Testing (Electron)

QA agents can test the UI via Electron MCP:
```python
# Enable in .env: ELECTRON_MCP_ENABLED=true
# Start app: npm run dev (port 9222)
# QA agents get electron MCP tools automatically
```

## File Locations

```
apps/backend/
  ├── core/           # Client, auth, security
  ├── agents/         # Agent implementations
  ├── prompts/        # Agent system prompts
  └── integrations/   # Graphiti, Linear, GitHub

apps/frontend/
  ├── src/main/       # Electron main process
  ├── src/renderer/   # React UI components
  └── src/shared/     # Types, constants, i18n

tests/                # Test suite (pytest)
```

## Key Rules

1. **Never use `anthropic.Anthropic()` directly** - use `create_client()` from `core.client`
2. **All frontend text uses i18n** - `t('namespace:key')`, never hardcoded strings
3. **All PRs target `develop`** - not `main`
4. **Type hints required** - Python functions need types, TS strict mode
5. **Pre-commit hooks must pass** - ruff, eslint, typecheck, tests
6. **Named exports preferred** - avoid default exports in TS/React
7. **Tests required** - new features need tests, bugs need regression tests

## Quick Reference

| Task | Command |
|------|---------|
| Run single Python test | `apps/backend/.venv/bin/pytest tests/test_file.py::test_name -v` |
| Run all Python tests | `npm run test:backend` |
| Run frontend tests | `cd apps/frontend && npm test` |
| Lint Python | `cd apps/backend && ruff check --fix .` |
| Lint TypeScript | `cd apps/frontend && npm run lint:fix` |
| Type check | `cd apps/frontend && npm run typecheck` |
| Run pre-commit | `pre-commit run --all-files` |
| Start dev server | `npm run dev` |

For detailed architecture, see [CLAUDE.md](CLAUDE.md).
For contribution workflow, see [CONTRIBUTING.md](CONTRIBUTING.md).
