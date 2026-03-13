# Contributing Guide

This document defines the development standards, git workflow, and conventions for WorldRoot. Following these consistently keeps the codebase clean and makes the project portfolio-ready at all times.


## Git Workflow

### Branch Strategy

We use a simplified Git Flow model:

```
main              -- Production-ready code. Always deployable.
  └── develop     -- Integration branch. Features merge here first.
       ├── feature/codex-search      -- Feature branches
       ├── feature/crafting-engine
       ├── fix/auth-token-refresh
       └── chore/docker-config
```

**Branch naming conventions:**
- `feature/<short-description>` -- New features
- `fix/<short-description>` -- Bug fixes
- `chore/<short-description>` -- Config, dependency updates, docs, refactors
- `test/<short-description>` -- Adding or improving tests

### Commit Messages

Follow the Conventional Commits specification. This is the standard most teams use, and many CI tools can parse it automatically.

```
<type>(<scope>): <short description>

<optional body>
```

**Types:**
- `feat` -- A new feature
- `fix` -- A bug fix
- `docs` -- Documentation only
- `style` -- Formatting, missing semicolons, etc. (not CSS changes)
- `refactor` -- Code change that neither fixes a bug nor adds a feature
- `test` -- Adding or correcting tests
- `chore` -- Build process, dependencies, config

**Examples:**
```
feat(codex): add full-text search endpoint with tsvector
fix(auth): handle expired refresh token gracefully
docs(readme): add setup instructions for Docker
refactor(workshop): extract byproduct logic into dedicated service
test(crafting): add unit tests for recipe matching engine
chore(deps): bump FastAPI to 0.110.0
```

**Rules:**
- Use the imperative mood ("add feature" not "added feature")
- Keep the first line under 72 characters
- Reference issue numbers in the body if applicable

### Pull Request Process

Even working solo, use PRs for every feature. This builds the habit and creates a reviewable history.

1. Create a feature branch from `develop`
2. Make your changes with atomic, well-described commits
3. Push the branch and open a PR against `develop`
4. Write a brief PR description covering: what changed, why, and how to test it
5. Review your own diff before merging (seriously, this catches things)
6. Squash merge into `develop` to keep the history clean
7. Delete the feature branch after merge


## Code Standards

### Python (Backend)

**Style:**
- Follow PEP 8
- Use type hints on all function signatures and return types
- Maximum line length: 100 characters
- Use `ruff` for linting and formatting (configured in `pyproject.toml`)

**Naming:**
- Files: `snake_case.py`
- Classes: `PascalCase`
- Functions and variables: `snake_case`
- Constants: `UPPER_SNAKE_CASE`
- Private methods/attributes: prefix with single underscore `_method_name`

**Imports:**
- Standard library first, then third-party, then local. Separated by blank lines.
- Use absolute imports from the `app` package.

```python
import os
from datetime import datetime

from fastapi import APIRouter, Depends, HTTPException
from sqlalchemy.ext.asyncio import AsyncSession

from app.dependencies import get_db, get_current_user
from app.services.codex_service import CodexService
```

**FastAPI Conventions:**
- Route handlers should be thin. Validate input, call a service, return a response.
- Business logic lives in the `services/` layer, never in route handlers.
- Use dependency injection for database sessions and auth.
- All request/response bodies use Pydantic schemas, never raw dicts.

```python
# Good: thin route handler
@router.post("/", response_model=EntryResponse)
async def create_entry(
    data: EntryCreate,
    db: AsyncSession = Depends(get_db),
    user: User = Depends(require_dm),
):
    service = CodexService(db)
    entry = await service.create_entry(data, user)
    return entry

# Bad: business logic in the route
@router.post("/")
async def create_entry(data: EntryCreate, db: AsyncSession = Depends(get_db)):
    entry = CodexEntry(**data.dict())
    db.add(entry)
    await db.commit()
    # ... validation, permission checks, etc all crammed in here
```

**Error Handling:**
- Raise `HTTPException` in route handlers for HTTP errors
- Raise custom exceptions in services, catch them in routes
- Never return bare 500 errors. Always handle or log.

### TypeScript (Frontend)

**Style:**
- Use functional components exclusively, no class components
- Use named exports for components, default exports only for page-level components
- Use `const` by default, `let` only when reassignment is needed, never `var`
- Use strict TypeScript. No `any` types unless absolutely unavoidable and commented.

**Naming:**
- Component files: `PascalCase.tsx`
- Hook files: `camelCase.ts` (e.g., `useAuth.ts`)
- Utility files: `camelCase.ts`
- Type files: `camelCase.ts` in `types/` directory
- Interfaces/Types: `PascalCase` (e.g., `CodexEntry`, `CraftingAttempt`)

**Component Structure:**
```tsx
// 1. Imports
import { useState } from "react";
import { useQuery } from "@tanstack/react-query";

// 2. Types (if component-specific)
interface EntryCardProps {
  entry: CodexEntry;
  isLocked: boolean;
}

// 3. Component
export function EntryCard({ entry, isLocked }: EntryCardProps) {
  // Hooks first
  const [expanded, setExpanded] = useState(false);

  // Derived state
  const displayTitle = isLocked ? "???" : entry.title;

  // Handlers
  const handleClick = () => {
    if (!isLocked) setExpanded(!expanded);
  };

  // Render
  return (
    <div onClick={handleClick}>
      <h3>{displayTitle}</h3>
      {expanded && <p>{entry.teaser}</p>}
    </div>
  );
}
```

**State Management Rules:**
- Server state (data from API): TanStack Query. Always.
- Client-only UI state (modals, filters, workbench slots): Zustand or local useState.
- Never duplicate server state in Zustand. That is what React Query's cache is for.

**API Calls:**
- All API calls live in `src/api/` as standalone async functions.
- Components never call `fetch` or `axios` directly. They use hooks that wrap React Query.

```typescript
// src/api/codex.ts
export async function getEntries(campaignId: string): Promise<CodexEntry[]> {
  const response = await client.get(`/campaigns/${campaignId}/codex`);
  return response.data;
}

// src/hooks/useCodex.ts
export function useEntries(campaignId: string) {
  return useQuery({
    queryKey: ["codex", campaignId],
    queryFn: () => getEntries(campaignId),
  });
}

// Component just uses the hook
const { data: entries, isLoading } = useEntries(campaignId);
```

### SQL and Database

- Table names: `snake_case`, plural (`users`, `codex_entries`, `crafting_attempts`)
- Column names: `snake_case`
- Foreign keys: `<referenced_table_singular>_id` (e.g., `user_id`, `campaign_id`)
- Always include `created_at` and `updated_at` on tables that change
- Use UUIDs for primary keys (better for distributed systems and avoids sequential ID enumeration)
- Write migrations with Alembic, never modify the database schema by hand

### CSS / Tailwind

- Use Tailwind utility classes as the primary styling method
- Extract repeated patterns into reusable components, not custom CSS classes
- Use Tailwind's `@apply` directive sparingly, only for truly global patterns
- Theme tokens (colors, fonts, spacing) are defined in `tailwind.config.ts`
- Fantasy theme colors and fonts are configured there, not hardcoded in components


## File Organization Rules

- One component per file
- Keep files under 200 lines. If a component exceeds this, extract sub-components.
- Co-locate tests with source files or in a parallel `__tests__` directory
- Group by domain (codex, workshop, auth) not by type (all components together, all hooks together)


## Testing

### Backend
- Use `pytest` with `pytest-asyncio` for async test support
- Test the service layer primarily, not individual route handlers
- Use a test database (separate from dev) that resets between test runs
- Fixtures for common objects (test user, test campaign, test materials)

### Frontend
- Use Vitest (pairs with Vite) and React Testing Library
- Test component behavior, not implementation details
- Focus on: does the right content render? Do interactions trigger the right state changes?
- Mock API calls at the hook level, not inside components


## Environment Variables

- Never commit secrets or credentials
- Use `.env` for local development (gitignored)
- Document every variable in `.env.example` with placeholder values and comments
- Access via `pydantic-settings` on the backend, `import.meta.env` on the frontend
