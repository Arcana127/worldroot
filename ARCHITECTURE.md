# WorldRoot -- Architecture & Technical Planning Document

## Project Overview

WorldRoot is a unified web platform for the Avanra D&D campaign that combines two core modules:

1. **The Codex** -- A living, searchable encyclopedia of Avanra with fog-of-war mechanics. Players explore and unlock lore entries as they progress through the campaign. Reading and engaging with lore earns in-game rewards.

2. **The Workshop** -- A crafting system where players combine materials gathered during sessions to forge items between games. Recipes are discovered through experimentation, and every attempt yields something, even if it is not what the player expected.

Both modules share a unified authentication system, player profiles, a DM admin panel, and a notification layer.


---


## Tech Stack

### Frontend
- **React 18** with **TypeScript** -- TypeScript is non-negotiable for portfolio work in 2025/2026. Every major employer expects it, and it makes your codebase dramatically easier to reason about at scale.
- **React Router v6** -- Client-side routing for the SPA.
- **TanStack Query (React Query)** -- Server state management. Handles caching, background refetching, and loading/error states so you are not hand-rolling that logic in every component.
- **Tailwind CSS** -- Utility-first styling. Fast to iterate on, easy to theme (important for the fantasy aesthetic), and widely used in industry.
- **Zustand** -- Lightweight client state management for things like UI state, active filters, and the crafting workbench. Much simpler than Redux for a project this size.

### Backend
- **Python 3.12** with **FastAPI** -- FastAPI is the modern standard for Python web APIs. It is async-native, auto-generates OpenAPI/Swagger docs from your type hints, and has built-in request validation via Pydantic. Employers love seeing it because it signals you understand modern Python, not just Flask tutorials.
- **SQLAlchemy 2.0** (async) -- ORM with the new 2.0-style query API. Pairs naturally with FastAPI's async patterns.
- **Alembic** -- Database migrations. You will need this the moment your schema evolves, which it will.
- **Pydantic v2** -- Data validation and serialization. FastAPI uses it natively, so your request/response models are validated automatically.

### Database
- **PostgreSQL 16** -- The industry default for relational data. Your data is highly relational (users belong to campaigns, entries have unlock states per user, recipes reference materials, etc.), so Postgres is the right call over something like MongoDB.
- **Full-text search via pg_trgm + tsvector** -- PostgreSQL's built-in search is powerful enough for this use case. No need to add Elasticsearch complexity.

### Auth
- **JWT access tokens + refresh tokens** -- Stateless auth with short-lived access tokens (15 min) and longer-lived refresh tokens (7 days). Standard industry pattern.
- **bcrypt** for password hashing via `passlib`.
- Role-based access: `DM` (admin) and `Player` roles with middleware-level enforcement.

### DevOps / Tooling
- **Docker + Docker Compose** -- Containerized dev environment. One `docker compose up` spins up the frontend, backend, and database together.
- **GitHub Actions** -- CI pipeline for linting, type checking, and tests.
- **Vite** -- Frontend build tool. Fast dev server, fast builds, and the React ecosystem has largely moved here from Create React App.

### Why this stack specifically
This is not a "whatever works" selection. Every piece here maps to what hiring managers and senior engineers expect to see on resumes for mid-level full-stack and backend roles in 2026. FastAPI + TypeScript + PostgreSQL + Docker is a combination that signals professional-grade thinking rather than bootcamp defaults.


---


## System Architecture

```
┌─────────────────────────────────────────────────────────┐
│                        Client                           │
│                                                         │
│   React + TypeScript SPA (Vite)                         │
│   ┌──────────┐  ┌──────────┐  ┌──────────┐             │
│   │  Codex   │  │ Workshop │  │  Shared  │             │
│   │  Module  │  │  Module  │  │ (Auth,   │             │
│   │          │  │          │  │  Nav,    │             │
│   │          │  │          │  │  Notifs) │             │
│   └──────────┘  └──────────┘  └──────────┘             │
│                       │                                 │
└───────────────────────┼─────────────────────────────────┘
                        │ HTTP/REST (JSON)
                        ▼
┌─────────────────────────────────────────────────────────┐
│                    FastAPI Backend                       │
│                                                         │
│   ┌──────────┐  ┌──────────┐  ┌──────────┐             │
│   │  /codex  │  │ /workshop│  │  /auth   │             │
│   │  routes  │  │  routes  │  │  /users  │             │
│   └────┬─────┘  └────┬─────┘  └────┬─────┘             │
│        │              │              │                   │
│   ┌────┴──────────────┴──────────────┴─────┐            │
│   │         Service Layer (business logic)  │            │
│   └────────────────────┬───────────────────┘            │
│                        │                                 │
│   ┌────────────────────┴───────────────────┐            │
│   │      SQLAlchemy ORM + Alembic          │            │
│   └────────────────────┬───────────────────┘            │
│                        │                                 │
└────────────────────────┼────────────────────────────────┘
                         │
                         ▼
              ┌─────────────────────┐
              │   PostgreSQL 16     │
              │                     │
              │  users              │
              │  campaigns          │
              │  codex_entries      │
              │  entry_unlocks      │
              │  materials          │
              │  recipes            │
              │  crafting_attempts  │
              │  inventories        │
              │  notifications      │
              └─────────────────────┘
```


---


## Database Schema

### Core Tables

```sql
-- Campaign and user management
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    username VARCHAR(50) UNIQUE NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    role VARCHAR(20) NOT NULL DEFAULT 'player',  -- 'dm' or 'player'
    created_at TIMESTAMPTZ DEFAULT now(),
    updated_at TIMESTAMPTZ DEFAULT now()
);

CREATE TABLE campaigns (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(100) NOT NULL,
    dm_id UUID REFERENCES users(id) NOT NULL,
    invite_code VARCHAR(20) UNIQUE NOT NULL,
    created_at TIMESTAMPTZ DEFAULT now()
);

CREATE TABLE campaign_members (
    campaign_id UUID REFERENCES campaigns(id) ON DELETE CASCADE,
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    character_name VARCHAR(100),
    joined_at TIMESTAMPTZ DEFAULT now(),
    PRIMARY KEY (campaign_id, user_id)
);
```

### Codex Tables

```sql
CREATE TABLE codex_categories (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    campaign_id UUID REFERENCES campaigns(id) ON DELETE CASCADE,
    name VARCHAR(100) NOT NULL,        -- e.g. "Locations", "NPCs", "Factions", "History"
    icon VARCHAR(50),                  -- frontend icon identifier
    sort_order INTEGER DEFAULT 0
);

CREATE TABLE codex_entries (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    campaign_id UUID REFERENCES campaigns(id) ON DELETE CASCADE,
    category_id UUID REFERENCES codex_categories(id),
    title VARCHAR(200) NOT NULL,
    slug VARCHAR(200) NOT NULL,
    
    -- Content tiers for progressive reveal
    teaser TEXT,                        -- always visible once unlocked (1-2 sentences)
    summary TEXT,                       -- unlocked at tier 2 (a paragraph)
    full_content TEXT,                  -- unlocked at tier 3 (the full entry)
    dm_notes TEXT,                      -- never shown to players
    
    -- Metadata
    tags TEXT[],                        -- for search and cross-referencing
    related_entries UUID[],            -- links to other codex entries
    is_published BOOLEAN DEFAULT false, -- DM controls when entries go live
    created_at TIMESTAMPTZ DEFAULT now(),
    updated_at TIMESTAMPTZ DEFAULT now(),
    
    -- Full-text search
    search_vector TSVECTOR GENERATED ALWAYS AS (
        setweight(to_tsvector('english', coalesce(title, '')), 'A') ||
        setweight(to_tsvector('english', coalesce(teaser, '')), 'B') ||
        setweight(to_tsvector('english', coalesce(summary, '')), 'C') ||
        setweight(to_tsvector('english', coalesce(full_content, '')), 'D')
    ) STORED,
    
    UNIQUE(campaign_id, slug)
);

CREATE INDEX idx_codex_search ON codex_entries USING GIN(search_vector);
CREATE INDEX idx_codex_tags ON codex_entries USING GIN(tags);

-- Tracks what each player has unlocked and at what tier
CREATE TABLE entry_unlocks (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    entry_id UUID REFERENCES codex_entries(id) ON DELETE CASCADE,
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    tier INTEGER NOT NULL DEFAULT 1,   -- 1=teaser, 2=summary, 3=full
    unlocked_at TIMESTAMPTZ DEFAULT now(),
    unlocked_by VARCHAR(50),           -- 'dm_grant', 'exploration', 'research', 'reward'
    UNIQUE(entry_id, user_id)
);

-- Track lore engagement for reward system
CREATE TABLE lore_interactions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id),
    entry_id UUID REFERENCES codex_entries(id),
    interaction_type VARCHAR(30),       -- 'read', 'bookmarked', 'referenced_in_session'
    created_at TIMESTAMPTZ DEFAULT now()
);
```

### Workshop Tables

```sql
-- Materials that players collect during sessions
CREATE TABLE materials (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    campaign_id UUID REFERENCES campaigns(id) ON DELETE CASCADE,
    name VARCHAR(100) NOT NULL,
    description TEXT,
    rarity VARCHAR(20) DEFAULT 'common',  -- common, uncommon, rare, legendary
    
    -- Properties stored as JSONB for flexibility
    -- e.g. {"element": "fire", "hardness": 7, "magical": true, "volatile": false}
    properties JSONB NOT NULL DEFAULT '{}',
    
    icon VARCHAR(50),                    -- frontend icon/image identifier
    is_published BOOLEAN DEFAULT false
);

-- Player inventories of materials
CREATE TABLE player_inventories (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id),
    campaign_id UUID REFERENCES campaigns(id),
    material_id UUID REFERENCES materials(id),
    quantity INTEGER NOT NULL DEFAULT 0 CHECK (quantity >= 0),
    acquired_at TIMESTAMPTZ DEFAULT now(),
    source VARCHAR(100),                -- "session_12", "trade", "crafting_byproduct"
    UNIQUE(user_id, campaign_id, material_id)
);

-- Recipes define what combinations produce what results
CREATE TABLE recipes (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    campaign_id UUID REFERENCES campaigns(id) ON DELETE CASCADE,
    name VARCHAR(200) NOT NULL,
    description TEXT,
    result_type VARCHAR(50),            -- 'weapon', 'armor', 'potion', 'trinket', 'scroll'
    
    -- Ingredients as JSONB array
    -- e.g. [{"material_id": "...", "quantity": 2}, {"material_id": "...", "quantity": 1}]
    ingredients JSONB NOT NULL,
    
    -- The crafted item's stats/effects
    result_data JSONB NOT NULL,         -- flexible structure per result_type
    
    -- Discovery settings
    is_hidden BOOLEAN DEFAULT true,     -- hidden recipes must be discovered
    difficulty INTEGER DEFAULT 10,      -- DC for crafting check
    
    created_at TIMESTAMPTZ DEFAULT now()
);

-- Every crafting attempt is logged, successes and failures
CREATE TABLE crafting_attempts (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id),
    campaign_id UUID REFERENCES campaigns(id),
    recipe_id UUID REFERENCES recipes(id),  -- NULL if no matching recipe exists
    
    -- What they actually combined
    ingredients_used JSONB NOT NULL,
    
    -- Outcome
    success BOOLEAN NOT NULL,
    result_name VARCHAR(200),
    result_data JSONB,                  -- the item they got (could be a byproduct)
    roll_value INTEGER,                 -- what they rolled
    
    crafted_at TIMESTAMPTZ DEFAULT now()
);

-- Player's crafted items
CREATE TABLE player_items (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id),
    campaign_id UUID REFERENCES campaigns(id),
    crafting_attempt_id UUID REFERENCES crafting_attempts(id),
    name VARCHAR(200) NOT NULL,
    item_type VARCHAR(50),
    item_data JSONB NOT NULL,           -- stats, effects, description
    is_approved BOOLEAN DEFAULT false,  -- DM approval for use in-game
    created_at TIMESTAMPTZ DEFAULT now()
);

-- Discovered recipe journal (auto-populates on first successful craft)
CREATE TABLE recipe_journal (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id),
    recipe_id UUID REFERENCES recipes(id),
    campaign_id UUID REFERENCES campaigns(id),
    discovered_at TIMESTAMPTZ DEFAULT now(),
    times_crafted INTEGER DEFAULT 1,
    UNIQUE(user_id, recipe_id)
);
```

### Shared Tables

```sql
CREATE TABLE notifications (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    campaign_id UUID REFERENCES campaigns(id),
    type VARCHAR(50) NOT NULL,          -- 'lore_unlocked', 'craft_complete', 'dm_message', etc.
    title VARCHAR(200) NOT NULL,
    message TEXT,
    data JSONB,                         -- contextual data (entry_id, item_id, etc.)
    read BOOLEAN DEFAULT false,
    created_at TIMESTAMPTZ DEFAULT now()
);

CREATE TABLE activity_log (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id),
    campaign_id UUID REFERENCES campaigns(id),
    action VARCHAR(50) NOT NULL,
    details JSONB,
    created_at TIMESTAMPTZ DEFAULT now()
);
```


---


## API Design

### Auth Endpoints
```
POST   /api/auth/register          -- Create account
POST   /api/auth/login             -- Get JWT tokens
POST   /api/auth/refresh           -- Refresh access token
POST   /api/auth/logout            -- Invalidate refresh token
```

### Campaign Endpoints
```
POST   /api/campaigns              -- Create campaign (DM only)
GET    /api/campaigns              -- List user's campaigns
POST   /api/campaigns/join         -- Join via invite code
GET    /api/campaigns/:id          -- Campaign details + members
```

### Codex Endpoints
```
GET    /api/campaigns/:id/codex                     -- List all entries (respects fog-of-war)
GET    /api/campaigns/:id/codex/search?q=           -- Full-text search (only searches unlocked content)
GET    /api/campaigns/:id/codex/categories          -- List categories
GET    /api/campaigns/:id/codex/:slug               -- Single entry (returns content up to player's tier)
POST   /api/campaigns/:id/codex/:slug/interact      -- Log a read/bookmark interaction

-- DM-only
POST   /api/campaigns/:id/codex                     -- Create entry
PUT    /api/campaigns/:id/codex/:slug               -- Edit entry
POST   /api/campaigns/:id/codex/:slug/unlock        -- Unlock entry for player(s)
DELETE /api/campaigns/:id/codex/:slug               -- Remove entry
```

### Workshop Endpoints
```
GET    /api/campaigns/:id/materials                 -- List all known materials
GET    /api/campaigns/:id/inventory                 -- Player's material inventory
POST   /api/campaigns/:id/craft                     -- Attempt a craft (core gameplay loop)
GET    /api/campaigns/:id/recipes                   -- Player's discovered recipe journal
GET    /api/campaigns/:id/items                     -- Player's crafted items
GET    /api/campaigns/:id/craft/history             -- Crafting attempt log

-- DM-only
POST   /api/campaigns/:id/materials                 -- Create material
PUT    /api/campaigns/:id/materials/:id             -- Edit material
POST   /api/campaigns/:id/inventory/grant           -- Give materials to player(s)
POST   /api/campaigns/:id/recipes                   -- Create recipe
POST   /api/campaigns/:id/items/:id/approve         -- Approve crafted item for in-game use
```

### Notification Endpoints
```
GET    /api/notifications                           -- List notifications
PATCH  /api/notifications/:id/read                  -- Mark as read
POST   /api/notifications/read-all                  -- Mark all as read
```


---


## Frontend Architecture

### Route Structure
```
/                           -- Landing / dashboard
/login                      -- Auth
/register                   -- Auth
/campaigns                  -- Campaign selector
/campaigns/:id              -- Campaign home (activity feed)
/campaigns/:id/codex        -- Codex browser with category sidebar
/campaigns/:id/codex/:slug  -- Single entry view
/campaigns/:id/workshop     -- Crafting workbench
/campaigns/:id/workshop/journal    -- Discovered recipes
/campaigns/:id/workshop/inventory  -- Material inventory
/campaigns/:id/workshop/items      -- Crafted items
/campaigns/:id/admin        -- DM panel (DM only)
/campaigns/:id/admin/codex  -- Manage codex entries
/campaigns/:id/admin/workshop -- Manage materials, recipes, inventories
```

### Component Organization
```
src/
├── api/                     -- API client functions (one file per domain)
│   ├── auth.ts
│   ├── codex.ts
│   ├── workshop.ts
│   └── client.ts            -- Axios/fetch wrapper with auth interceptors
├── components/
│   ├── common/              -- Buttons, modals, search bars, loading states
│   ├── codex/
│   │   ├── EntryCard.tsx          -- Preview card with lock/unlock state
│   │   ├── EntryViewer.tsx        -- Full entry display with tier reveals
│   │   ├── CategorySidebar.tsx
│   │   ├── CodexSearch.tsx
│   │   └── FogOverlay.tsx         -- Visual fog effect on locked content
│   ├── workshop/
│   │   ├── CraftingWorkbench.tsx  -- Drag-and-drop material slots
│   │   ├── MaterialCard.tsx       -- Material with properties display
│   │   ├── InventoryGrid.tsx
│   │   ├── RecipeJournal.tsx
│   │   ├── CraftingResult.tsx     -- Animated result reveal
│   │   └── ItemCard.tsx
│   ├── admin/
│   │   ├── EntryEditor.tsx        -- Rich text editor for codex entries
│   │   ├── UnlockManager.tsx      -- Bulk unlock UI for players
│   │   ├── MaterialEditor.tsx
│   │   ├── RecipeBuilder.tsx      -- Visual recipe creation
│   │   └── InventoryManager.tsx   -- Grant/revoke materials
│   └── layout/
│       ├── AppShell.tsx
│       ├── Navbar.tsx
│       └── NotificationBell.tsx
├── hooks/                   -- Custom hooks (useAuth, useCodex, useCrafting, etc.)
├── pages/                   -- Route-level page components
├── stores/                  -- Zustand stores
├── types/                   -- TypeScript interfaces and types
├── utils/                   -- Helpers (formatting, crafting logic, etc.)
└── styles/                  -- Tailwind config, theme tokens, global styles
```


---


## Core Feature Logic

### Codex: Fog-of-War System

The fog-of-war mechanic is the heart of the Codex. Here is how it works:

1. The DM creates entries and assigns them to categories. Each entry has three content tiers: teaser, summary, and full.
2. By default, entries are invisible to players until the DM publishes them AND unlocks them for specific players.
3. When a player views the Codex, the API checks their `entry_unlocks` records and returns only the content they are authorized to see.
4. The frontend renders locked entries as silhouettes or fog-covered cards. Players can see that something exists in a category, but not what it is, creating curiosity.
5. The DM can unlock entries in bulk (e.g., "the party just visited Thalazzara, unlock all Thalazzara entries at tier 1 for everyone") or individually.
6. Tier upgrades happen through gameplay: initial discovery gives tier 1, researching in-game gives tier 2, and deep engagement gives tier 3.

The search system only indexes content the player has actually unlocked. A player searching for "Thalazzara" will not find entries they have not discovered yet, even if those entries contain the word.

### Workshop: Crafting Engine

The crafting loop works like this:

1. **Material collection** happens during sessions. After each session, the DM uses the admin panel to grant materials to players based on what they found.

2. **The workbench** is a drag-and-drop interface with 2-5 input slots. Players drag materials from their inventory into the slots and hit "Forge."

3. **Recipe matching** runs server-side. The backend checks the combination against the `recipes` table:
   - If an exact match exists and the player meets the difficulty check, they get the intended result. The recipe is added to their journal.
   - If an exact match exists but they fail the check, they get a "flawed" version of the item (reduced stats, cosmetic defect, etc.) and still learn that the recipe exists.
   - If no exact match exists, the system runs a **byproduct generator** based on the properties of the materials used. Fire + brittle materials might yield ash or cinders. Magical + regenerative might yield a weak healing salve. The player always gets something.

4. **Byproduct generation logic** (server-side):
   ```
   Input: list of material properties
   Process:
     - Collect all properties from input materials
     - Check for property interactions (fire + ice = steam, etc.)
     - Weight by rarity of materials used
     - Roll on a byproduct table seeded by the property combination
   Output: a named byproduct item with flavor text and minor mechanical effect
   ```

5. **Recipe discovery** is organic. Players are never shown the recipe list. They experiment, and when something works, it goes in their journal. The journal shows the ingredients and result but not the difficulty check, so there is still some mystery around consistency.

6. **DM approval** is optional but available. High-powered items can require DM sign-off before they are considered "canon" in the game.


---


## Backend Project Structure

```
backend/
├── alembic/                     -- Database migrations
│   └── versions/
├── app/
│   ├── __init__.py
│   ├── main.py                  -- FastAPI app entry point
│   ├── config.py                -- Environment config via pydantic-settings
│   ├── database.py              -- Async SQLAlchemy engine + session
│   ├── dependencies.py          -- Dependency injection (get_db, get_current_user, etc.)
│   ├── models/                  -- SQLAlchemy ORM models
│   │   ├── user.py
│   │   ├── campaign.py
│   │   ├── codex.py
│   │   ├── workshop.py
│   │   └── notification.py
│   ├── schemas/                 -- Pydantic request/response schemas
│   │   ├── auth.py
│   │   ├── codex.py
│   │   ├── workshop.py
│   │   └── common.py
│   ├── routes/                  -- API route handlers
│   │   ├── auth.py
│   │   ├── campaigns.py
│   │   ├── codex.py
│   │   ├── workshop.py
│   │   └── notifications.py
│   ├── services/                -- Business logic layer
│   │   ├── auth_service.py
│   │   ├── codex_service.py
│   │   ├── crafting_engine.py   -- Core crafting logic + byproduct generation
│   │   ├── inventory_service.py
│   │   └── notification_service.py
│   └── utils/
│       ├── security.py          -- JWT + password hashing
│       └── permissions.py       -- Role-based access helpers
├── tests/
│   ├── conftest.py              -- Fixtures (test db, test client, mock users)
│   ├── test_auth.py
│   ├── test_codex.py
│   └── test_crafting.py
├── alembic.ini
├── requirements.txt
├── Dockerfile
└── .env.example
```


---


## Development Roadmap

### Phase 1: Foundation (Week 1-2)
- Project scaffolding (React + FastAPI + Docker Compose + PostgreSQL)
- Database schema + Alembic migrations
- Auth system (register, login, JWT, role middleware)
- Campaign CRUD + invite codes
- Basic frontend shell with routing and auth flow

### Phase 2: Codex MVP (Week 3-4)
- Codex CRUD endpoints (DM creates/edits entries)
- Fog-of-war query logic (filter by unlock state)
- Entry unlock system (DM grants access)
- Frontend: category browser, entry cards with lock/unlock visuals, entry viewer
- Full-text search (scoped to unlocked content)
- Lore interaction tracking

### Phase 3: Workshop MVP (Week 5-6)
- Material and recipe CRUD (DM-facing)
- Player inventory system
- Crafting engine: recipe matching + difficulty rolls
- Byproduct generator for failed/unmatched crafts
- Frontend: crafting workbench (drag-and-drop), inventory grid, recipe journal
- Crafting result animation/reveal

### Phase 4: Polish & Integration (Week 7-8)
- Notification system (lore unlocks, craft completions, DM messages)
- DM admin dashboard (bulk unlock, inventory grants, approval queue)
- Cross-module links (codex entries can reference craftable materials and vice versa)
- Lore engagement reward tracking (inspiration points, read streaks)
- Visual polish: fantasy theming, animations, responsive design
- Testing: unit tests for crafting engine, integration tests for API

### Phase 5: Portfolio Ready (Week 9-10)
- README with screenshots, architecture diagram, and feature walkthrough
- Demo seed data (pre-populated Avanra content so reviewers can explore)
- Deployment (Railway, Render, or Fly.io for backend; Vercel or Netlify for frontend)
- Performance: query optimization, pagination, lazy loading
- Optional stretch: WebSocket notifications for real-time DM pushes


---


## Key Technical Decisions and Tradeoffs

**Why REST over GraphQL?**
REST is simpler to implement, debug, and explain in interviews. GraphQL adds complexity that is not justified by this data shape. The Codex search and Workshop crafting endpoints are straightforward request/response patterns, not deeply nested graph queries.

**Why JSONB for material properties and recipe data?**
The shape of item stats, material properties, and crafting results varies wildly depending on item type. A potion has duration and effect. A sword has damage dice and modifiers. A trinket might have a passive aura. JSONB lets you store these flexibly without a sprawling EAV (entity-attribute-value) table mess while still keeping the relational structure where it matters (users, campaigns, inventories, unlock states).

**Why not a game engine or WebSocket-heavy approach?**
This is a between-sessions companion tool, not a real-time game. HTTP request/response is the right model for 95% of interactions. The only place WebSockets would genuinely help is real-time DM push notifications during a session, and that is a Phase 5 stretch goal, not a core requirement.

**Why Zustand over Redux?**
Redux is overkill here. Most of the app's state is server state managed by React Query. Zustand handles the small amount of client-only state (current workbench slots, UI toggles, search filters) without the boilerplate.

**Why separate service layer from routes?**
This is the pattern that will matter most in interviews. Keeping business logic in services and HTTP handling in routes makes the code testable, reusable, and easy to explain. The crafting engine in particular benefits from being a standalone, well-tested module.
