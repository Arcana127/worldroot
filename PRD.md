# WorldRoot -- Product Requirements Document

## 1. Problem Statement

Tabletop RPG campaigns suffer from a continuity problem. Between sessions (often 1-2 weeks apart), players forget lore, lose track of plot threads, and have no meaningful way to engage with the game world. DMs spend time recapping instead of advancing the story, and rich worldbuilding goes unappreciated because there is no persistent, accessible way to explore it.

On the crafting and downtime side, most groups either skip it entirely or handle it through awkward text-message exchanges. There is no structured, fun way for players to interact with game systems outside of session time.


## 2. Target Users

### The Dungeon Master (Admin)
- Manages campaign content, lore, materials, and recipes
- Wants tools that reduce session prep, not add to it
- Needs granular control over what players see and when
- Values the ability to turn player engagement into plot hooks

### The Player
- Wants to stay connected to the campaign between sessions
- Enjoys exploring lore and feeling like discoveries are earned
- Likes crafting and experimentation mechanics
- Values getting tangible in-game rewards for between-session engagement


## 3. Success Metrics

- Players voluntarily log in between sessions at least once per week
- DMs spend less than 15 minutes per session managing content in the admin panel
- At least 70% of crafting attempts are initiated by players without DM prompting
- Players reference Codex entries during sessions organically


## 4. Feature Requirements

---

### 4.1 Authentication and Campaign Management

**User Stories**

As a player, I want to create an account and join my DM's campaign with a code so that I can access the platform quickly without a complicated onboarding process.

As a DM, I want to create a campaign and get a shareable invite code so that my players can join without me manually adding each one.

As a DM, I want to manage multiple campaigns from one account so that I do not need separate logins for each game I run.

**Acceptance Criteria**
- Users can register with username, email, and password
- Login returns JWT access token (15 min expiry) and refresh token (7 day expiry)
- DMs can create campaigns and receive a unique invite code
- Players can join a campaign by entering the invite code
- A user can belong to multiple campaigns
- A user can be a DM in one campaign and a player in another
- DM role is assigned per-campaign, not globally

---

### 4.2 The Codex

#### 4.2.1 Entry Management (DM)

**User Stories**

As a DM, I want to create lore entries with multiple content tiers so that I can control how much detail players receive at each stage of discovery.

As a DM, I want to write private notes on entries that only I can see so that I can track plot connections and secrets without exposing them.

As a DM, I want to organize entries into categories so that the Codex feels structured and browsable.

**Acceptance Criteria**
- DM can create entries with title, teaser, summary, full content, and DM notes
- DM can assign entries to categories (Locations, NPCs, Factions, History, Creatures, Items, Custom)
- DM can add tags for cross-referencing
- DM can link related entries to each other
- DM can publish/unpublish entries (unpublished entries are invisible to players entirely)
- DM can edit any field of an existing entry
- DM can delete entries

#### 4.2.2 Fog-of-War System

**User Stories**

As a DM, I want to unlock lore entries for specific players at specific tiers so that discovery feels personalized and earned.

As a DM, I want to bulk-unlock entries so that after a major story beat I can reveal a batch of lore quickly.

As a player, I want to see that locked entries exist without being able to read them so that I know there is more to discover.

**Acceptance Criteria**
- Players see locked entries as silhouettes/fog cards showing only the category and a placeholder
- Unlocking grants access at a specific tier (1=teaser, 2=summary, 3=full)
- DM can unlock entries for individual players or the entire party
- DM can upgrade a player's tier on an already-unlocked entry
- Unlock reasons are tracked (dm_grant, exploration, research, reward) for DM analytics
- The Codex landing page shows a visual ratio of unlocked vs total entries per category so players can gauge how much they have explored

#### 4.2.3 Search

**User Stories**

As a player, I want to search the Codex by keyword so that I can quickly find information my character would know.

**Acceptance Criteria**
- Full-text search across title, teaser, summary, and full content
- Search results are scoped to the player's unlocked content only. A player cannot discover entries through search that they have not unlocked.
- Search results are ranked by relevance (title matches weighted highest)
- Tag-based filtering alongside text search
- Results show the entry title and a snippet of the matching content

#### 4.2.4 Player Engagement

**User Stories**

As a player, I want to bookmark entries so that I can quickly reference important lore during sessions.

As a DM, I want to see which players are reading lore so that I can reward engagement and tailor content.

**Acceptance Criteria**
- Players can bookmark entries for quick access
- Read interactions are logged with timestamps
- DM dashboard shows engagement stats: most-read entries, most active readers, entries with zero reads
- Engagement data can inform in-game rewards (Inspiration, minor items, etc.) at DM discretion

---

### 4.3 The Workshop

#### 4.3.1 Materials and Inventory (DM)

**User Stories**

As a DM, I want to create materials with custom properties so that the crafting system reflects my world's unique resources.

As a DM, I want to grant materials to players after sessions so that their in-game loot feeds into the crafting system.

**Acceptance Criteria**
- DM can create materials with name, description, rarity, and a flexible property set (element, hardness, magical affinity, volatility, weight, etc.)
- Properties are stored as key-value pairs so the system is extensible
- DM can grant materials to individual players or the whole party in specified quantities
- DM can revoke materials if needed (e.g., correcting a mistake)
- Materials have a published/draft state so the DM can prepare them before revealing

#### 4.3.2 Player Inventory

**User Stories**

As a player, I want to view my material inventory so that I know what I have to work with.

**Acceptance Criteria**
- Inventory displays all materials with quantities, properties, and rarity
- Sorting by name, rarity, quantity, or property
- Filtering by property (show me all fire-aligned materials)
- Material cards show relevant properties at a glance
- Inventory updates in real-time when materials are used in crafting

#### 4.3.3 Crafting Workbench

**User Stories**

As a player, I want to combine materials on a workbench to try to craft items so that I can experiment and discover recipes.

As a player, I want to always get something from a crafting attempt so that experimenting never feels like a total waste.

**Acceptance Criteria**
- Workbench has 2-5 drag-and-drop input slots
- Players drag materials from their inventory into slots
- A "Forge" button submits the combination to the server
- Materials are consumed from inventory on submission regardless of outcome
- Three possible outcomes:
  1. Recipe match + success: player receives the intended item, recipe added to journal
  2. Recipe match + failure: player receives a flawed version, recipe is hinted at in journal
  3. No recipe match: byproduct generator creates a minor item based on material properties
- Result is displayed with an animated reveal (builds anticipation)
- Crafting history log shows all past attempts with ingredients and outcomes

#### 4.3.4 Byproduct Generator

**User Stories**

As a player, I want failed experiments to produce interesting byproducts so that I am encouraged to keep experimenting rather than only using known recipes.

**Acceptance Criteria**
- Byproducts are generated based on the combined properties of input materials
- Property interactions follow a defined interaction table (fire + ice = steam, magical + volatile = unstable essence, etc.)
- Byproducts have names, descriptions, and minor mechanical effects
- Rarer input materials increase the chance of useful byproducts
- Byproducts are added to the player's item collection
- The generation is deterministic enough that the same inputs lean toward similar outputs, rewarding player intuition

#### 4.3.5 Recipe Journal

**User Stories**

As a player, I want a journal that automatically records recipes I have discovered so that I can repeat successful crafts.

**Acceptance Criteria**
- Journal auto-populates when a player successfully crafts a known recipe
- Each journal entry shows the ingredients and the result
- Failed attempts against a known recipe add a "partial" entry hinting at the recipe
- Journal tracks how many times each recipe has been crafted
- Journal does not reveal recipes the player has not discovered, there is no browse-all-recipes view

#### 4.3.6 DM Approval

**User Stories**

As a DM, I want to approve or reject powerful crafted items before they enter the game so that the crafting system does not break game balance.

**Acceptance Criteria**
- DM can flag recipes as requiring approval
- Items crafted from flagged recipes enter a pending state
- DM sees a queue of pending items with full details
- DM can approve (item becomes usable), reject (item is removed, materials optionally refunded), or modify (adjust stats before approving)
- Players are notified of the decision

---

### 4.4 Notifications

**User Stories**

As a player, I want to be notified when new lore is unlocked for me so that I know to check the Codex.

As a player, I want to be notified when a crafting result is approved or rejected so that I know the status of my items.

**Acceptance Criteria**
- Notification bell in the navbar with unread count
- Notification types: lore_unlocked, craft_complete, item_approved, item_rejected, dm_message, material_granted
- Clicking a notification navigates to the relevant content
- Mark as read (individual and bulk)
- Notifications are persisted in the database, not ephemeral

---

### 4.5 DM Admin Dashboard

**User Stories**

As a DM, I want a centralized admin panel so that I can manage all aspects of the campaign without navigating through player-facing views.

**Acceptance Criteria**
- Dashboard landing shows campaign stats: active players, total codex entries, unlock progress, crafting activity
- Codex management: create, edit, publish, delete entries. Bulk unlock tool.
- Workshop management: create materials, create recipes, manage inventories, approval queue
- Player overview: per-player stats on engagement, inventory, crafting history
- Activity log showing recent platform actions

---

## 5. Non-Functional Requirements

### Performance
- API responses under 200ms for standard CRUD operations
- Full-text search under 500ms for up to 1000 entries
- Frontend initial load under 3 seconds on a standard connection

### Security
- Passwords hashed with bcrypt (minimum 12 rounds)
- JWT tokens with short expiry and refresh rotation
- All endpoints enforce role-based access at the middleware level
- Players cannot access other players' inventories or unlock states
- SQL injection prevention via parameterized queries (SQLAlchemy)
- Input validation on all endpoints via Pydantic schemas

### Accessibility
- Semantic HTML throughout
- Keyboard navigable interface
- Color contrast compliant with WCAG 2.1 AA
- Screen reader friendly labels on interactive elements

### Scalability
- Designed for single-campaign use initially (1 DM, 4-6 players)
- Schema and API support multi-campaign from day one
- No architectural changes needed to support 50+ concurrent campaigns


## 6. Out of Scope (For Now)

These are features that could be added later but are not part of the initial build:

- Real-time WebSocket updates (push notifications during live sessions)
- Player-to-player trading of materials or items
- Image uploads for codex entries or material icons (will use icon identifiers initially)
- Mobile native app (responsive web covers mobile use)
- AI-generated lore or crafting descriptions
- Voice integration or virtual tabletop (VTT) integration
- Public campaign sharing or community features
