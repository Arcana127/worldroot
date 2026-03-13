# WorldRoot

A companion web platform for tabletop RPG campaigns that brings the world to life between sessions. Players explore a living encyclopedia of campaign lore and experiment with a hands-on crafting system, all managed by the DM through an intuitive admin panel.

Built as a unified platform with two core modules:

**The Codex** -- A searchable, fog-of-war encyclopedia where lore entries are progressively revealed as players make discoveries in-game. Players only see what their characters have learned, creating a sense of genuine exploration even outside of sessions.

**The Workshop** -- A crafting system where players combine collected materials to forge weapons, potions, armor, and trinkets. Recipes are discovered through experimentation rather than handed out, and every attempt produces something, even when it goes sideways.


## Why This Exists

Tabletop RPG groups often lose momentum between sessions. Players forget plot threads, lore goes unread, and downtime activities get handwaved. WorldRoot solves this by giving players meaningful things to do between games that feed directly back into the campaign.

For DMs, it is a content management tool. For players, it is part reference guide and part minigame.


## Features

**Codex Module**
- Tiered content reveal (teaser, summary, full entry) controlled by the DM
- Fog-of-war browsing where locked entries appear as silhouettes
- Full-text search scoped to only content the player has unlocked
- Category organization (Locations, NPCs, Factions, History, etc.)
- Cross-referencing between related entries
- Lore engagement tracking for in-game reward eligibility

**Workshop Module**
- Drag-and-drop crafting workbench
- Material inventory management with properties system (elemental alignment, hardness, magical affinity, etc.)
- Recipe discovery through experimentation, not menus
- Byproduct generation for unmatched or failed combinations so nothing feels wasted
- Auto-populating recipe journal as players discover working combinations
- DM approval queue for high-powered crafted items

**Platform**
- Campaign management with invite codes
- Role-based access (DM and Player roles)
- Notification system for lore unlocks, crafting results, and DM messages
- DM admin dashboard for managing content, inventories, and player progress
- Responsive design for mobile use during sessions


## Tech Stack

| Layer | Technology | Why |
|-------|-----------|-----|
| Frontend | React 18, TypeScript, Vite | Industry standard SPA stack with type safety |
| Styling | Tailwind CSS | Utility-first, easy to theme for fantasy aesthetic |
| State (server) | TanStack Query | Handles caching, refetching, and loading states |
| State (client) | Zustand | Lightweight store for UI state and workbench slots |
| Backend | Python 3.12, FastAPI | Async-native, auto-generated API docs, Pydantic validation |
| ORM | SQLAlchemy 2.0 (async) | Modern Python ORM with full async support |
| Migrations | Alembic | Schema versioning and evolution |
| Database | PostgreSQL 16 | Relational data model with full-text search via tsvector |
| Auth | JWT (access + refresh tokens) | Stateless authentication with role-based middleware |
| DevOps | Docker, Docker Compose | One-command local development environment |
| CI | GitHub Actions | Automated linting, type checking, and tests |


## Quick Start

See [docs/SETUP.md](docs/SETUP.md) for detailed setup instructions (includes Windows/WSL2 guide).

```bash
# Clone the repo
git clone https://github.com/yourusername/worldroot.git
cd worldroot

# Copy environment config
cp .env.example .env

# Start everything
docker compose up --build

# Frontend:  http://localhost:5173
# Backend:   http://localhost:8000
# API Docs:  http://localhost:8000/docs
# Database:  localhost:5432
```


## Project Structure

```
worldroot/
├── frontend/               # React + TypeScript SPA
│   ├── src/
│   │   ├── api/            # API client functions
│   │   ├── components/     # UI components by domain
│   │   ├── hooks/          # Custom React hooks
│   │   ├── pages/          # Route-level page components
│   │   ├── stores/         # Zustand state stores
│   │   ├── types/          # TypeScript interfaces
│   │   └── utils/          # Helper functions
│   └── ...
├── backend/                # FastAPI application
│   ├── app/
│   │   ├── models/         # SQLAlchemy ORM models
│   │   ├── schemas/        # Pydantic request/response schemas
│   │   ├── routes/         # API endpoint handlers
│   │   ├── services/       # Business logic layer
│   │   └── utils/          # Auth, permissions, helpers
│   ├── alembic/            # Database migrations
│   └── tests/              # Backend test suite
├── docs/                   # Project documentation
├── docker-compose.yml
└── .env.example
```


## Documentation

- [Architecture Document](docs/ARCHITECTURE.md) -- System design, database schema, API design, and technical decisions
- [Product Requirements](docs/PRD.md) -- Feature specifications, user stories, and acceptance criteria
- [Contributing Guide](docs/CONTRIBUTING.md) -- Code standards, git workflow, and development conventions
- [Setup Guide](docs/SETUP.md) -- Local development environment setup (Windows/WSL2 + Docker)


## Development Roadmap

| Phase | Focus | Timeline |
|-------|-------|----------|
| 1 | Foundation (auth, campaigns, project scaffolding) | Week 1-2 |
| 2 | Codex MVP (entries, fog-of-war, search) | Week 3-4 |
| 3 | Workshop MVP (crafting engine, inventory, recipes) | Week 5-6 |
| 4 | Polish and integration (notifications, admin tools, theming) | Week 7-8 |
| 5 | Portfolio ready (deployment, seed data, README screenshots) | Week 9-10 |


## License

MIT
