# ⚔️ Real-Life RPG Task Manager — Personal Dev Notes

> Personal project log. Not public-facing yet.

---

## What I'm Building

A task manager that works like an RPG. Every task = a quest. Complete quests = earn XP, level up, get loot drops. Miss deadlines = lose HP. Built with HTML, CSS, JS (frontend) and PHP + MySQL (backend) with a clean client-server split.

The character system uses **Standing Stones** (inspired by Skyrim) instead of locked classes — your focus can change based on what you're working on in life right now.

---

## My Stack

- **Frontend:** HTML5, CSS3, JavaScript (UI logic, animations, fetch calls)
- **Backend:** PHP 8.x (game logic, XP calculations, loot rolls — never trust the client)
- **Database:** SQLite (local dev) → MySQL (when I move to a server)
- **Local server:** XAMPP
- **API:** REST, JSON over HTTP

### JS vs PHP — what goes where

This is important to get right from the start.

**PHP handles (server-side — the source of truth):**
- XP calculation and awarding
- Level-up checks
- Loot roll and drop logic
- HP changes from missed deadlines
- All database reads and writes
- Authentication and session management

**JS handles (client-side — UI only):**
- XP bar fill animation after a quest is completed
- Level-up animation and notification
- Quest board rendering without full page reloads
- Live form validation (before sending to PHP)
- Stone switcher UI interactions
- Filtering and sorting quests on the page

Rule: if it touches data or affects the character in any permanent way, it lives in PHP. JS is just for making things look and feel good.

---

## Folder Structure

```
rpg-task-manager/
│
├── client/                  # Everything the browser sees
│   ├── index.html           # Quest board
│   ├── character.html       # My character + active stone
│   ├── inventory.html       # My loot
│   ├── login.html
│   ├── css/
│   │   ├── main.css
│   │   ├── character.css
│   │   └── inventory.css
│   └── js/
│       ├── api.js           # All fetch() calls to the PHP API
│       ├── quests.js        # Quest board rendering + filtering
│       ├── character.js     # XP bar animation, level-up screen
│       ├── stones.js        # Stone switcher UI
│       └── inventory.js     # Loot display
│
├── server/                  # PHP backend
│   ├── index.php            # Router — all requests come through here
│   ├── config/
│   │   └── database.php     # DB connection (reads from .env)
│   ├── controllers/
│   │   ├── QuestController.php     # Create, complete, delete quests
│   │   ├── CharacterController.php # Stats, stone changes
│   │   ├── InventoryController.php
│   │   └── AuthController.php
│   ├── models/
│   │   ├── Quest.php
│   │   ├── Character.php
│   │   └── User.php
│   ├── game/
│   │   ├── XPEngine.php     # All XP and leveling calculations
│   │   ├── LootEngine.php   # Loot roll logic
│   │   └── HPEngine.php     # HP gain/loss logic
│   └── middleware/
│       └── auth.php         # JWT verification
│
├── database/
│   ├── schema.sql           # Run this first
│   └── seed.sql             # Sample quests and items for testing
│
├── .env                     # Local credentials — never commit
├── .env.example             # Safe version to commit
├── .gitignore
└── README.md
```

---

## Game Mechanics

### Quest Difficulties & XP

| Difficulty | XP reward | HP loss if failed |
|------------|-----------|-------------------|
| Common | 25 XP | −5 HP |
| Rare | 75 XP | −15 HP |
| Epic | 200 XP | −30 HP |
| Legendary | 500 XP | −50 HP |

XP is always calculated and awarded by PHP. JS only animates the result.

---

### Standing Stones (focus system)

Instead of a locked class, your character has an active **Standing Stone** that gives an XP bonus to a certain type of task. You can switch stones at any time, but there's a **7-day cooldown** after switching (so it stays meaningful, not just gaming the system).

This lives in PHP — the cooldown is checked server-side before any stone change is accepted.

| Stone | Focus | XP bonus |
|-------|-------|----------|
| 🌟 The Warrior Stone | Physical tasks (gym, sport, outdoor) | ×1.5 XP |
| 📚 The Mage Stone | Mental tasks (study, reading, research) | ×1.5 XP |
| 🗡️ The Thief Stone | Creative tasks (art, music, writing) | ×1.5 XP |
| 🌿 The Steed Stone | Productivity (work tasks, deadlines) | ×1.5 XP |
| 🌙 The Lover Stone | Social tasks (people, relationships) | ×1.5 XP |
| ⚗️ The Apprentice Stone | Skill-building (coding, new skills) | ×1.5 XP |

All stones give ×1.0 XP on tasks outside their focus category — no penalty, just no bonus.

Stone switch flow:
1. Player picks a new stone in the UI (JS renders the switcher)
2. JS sends a PATCH request to `/api/character/stone`
3. PHP checks the cooldown — if < 7 days since last switch, returns an error
4. If allowed, PHP updates the active stone and records the switch timestamp
5. JS displays the new stone and updated cooldown timer

---

### Leveling Formula

```
XP needed for level N = 100 * (N ^ 1.5)
```

| Level | XP needed |
|-------|-----------|
| 2 | 200 |
| 5 | 1,118 |
| 10 | 3,162 |
| 20 | 8,944 |
| 50 | 35,355 |

Level-up check runs in `XPEngine.php` every time XP is awarded.

---

### Loot Drops

| Rarity | Drop chance |
|--------|-------------|
| Common | 60% |
| Rare | 25% |
| Epic | 12% |
| Legendary | 3% |

Legendary quests have a higher base chance. Loot is rolled in `LootEngine.php` after a quest is marked complete.

---

### Daily Dungeon

A special set of 3–5 quests that resets every day at midnight. Completing the full dungeon grants a bonus XP reward and a guaranteed loot drop. Reset is handled by a timestamp check in PHP on login — no cron job needed for now.

---

## API Routes

```
POST   /api/auth/register
POST   /api/auth/login
POST   /api/auth/logout

GET    /api/quests
POST   /api/quests
PATCH  /api/quests/{id}/complete
DELETE /api/quests/{id}

GET    /api/character
PATCH  /api/character/stone      # Switch active standing stone

GET    /api/inventory

GET    /api/dungeon              # Today's dungeon quests
POST   /api/dungeon/complete     # Mark full dungeon as complete
```

---

## Database Schema

```sql
CREATE TABLE users (
    id INT AUTO_INCREMENT PRIMARY KEY,
    username VARCHAR(50) NOT NULL UNIQUE,
    email VARCHAR(100) NOT NULL UNIQUE,
    password_hash VARCHAR(255) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE characters (
    id INT AUTO_INCREMENT PRIMARY KEY,
    user_id INT NOT NULL,
    active_stone VARCHAR(50) DEFAULT 'warrior',
    stone_switched_at TIMESTAMP NULL,     -- for cooldown check
    level INT DEFAULT 1,
    xp INT DEFAULT 0,
    hp INT DEFAULT 100,
    max_hp INT DEFAULT 100,
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE
);

CREATE TABLE quests (
    id INT AUTO_INCREMENT PRIMARY KEY,
    user_id INT NOT NULL,
    title VARCHAR(255) NOT NULL,
    difficulty ENUM('common','rare','epic','legendary') DEFAULT 'common',
    category ENUM('physical','mental','creative','social','productivity','skill') DEFAULT 'mental',
    status ENUM('active','completed','failed') DEFAULT 'active',
    deadline DATETIME NULL,
    xp_reward INT NOT NULL,
    completed_at TIMESTAMP NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE
);

CREATE TABLE items (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    rarity ENUM('common','rare','epic','legendary') NOT NULL,
    description TEXT,
    icon VARCHAR(100)
);

CREATE TABLE inventory (
    id INT AUTO_INCREMENT PRIMARY KEY,
    user_id INT NOT NULL,
    item_id INT NOT NULL,
    obtained_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
    FOREIGN KEY (item_id) REFERENCES items(id)
);

CREATE TABLE daily_dungeon (
    id INT AUTO_INCREMENT PRIMARY KEY,
    dungeon_date DATE NOT NULL UNIQUE,
    quest_ids JSON NOT NULL,              -- array of quest IDs for this dungeon
    bonus_xp INT DEFAULT 150
);

CREATE TABLE dungeon_completions (
    id INT AUTO_INCREMENT PRIMARY KEY,
    user_id INT NOT NULL,
    dungeon_date DATE NOT NULL,
    completed_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE
);
```

---

## Development Phases

---

### Phase 1 — Local build (XAMPP, SQLite)

Goal: get the full app working on my own machine. No deployment, no live server, no stress. Just build and learn.

**Setup:**
- [ ] Install XAMPP, confirm Apache + MySQL work
- [ ] Create folder structure
- [ ] Set up `.env` and `.gitignore` (never commit `.env`)
- [ ] Write `schema.sql` and `seed.sql`, import into MySQL via phpMyAdmin

**Backend — PHP:**
- [ ] `index.php` router (parse URL and method, call the right controller)
- [ ] `database.php` connection file
- [ ] `AuthController.php` — register, login, logout with password_hash()
- [ ] `auth.php` middleware — JWT check on protected routes
- [ ] `QuestController.php` — full CRUD + complete action
- [ ] `XPEngine.php` — XP award, level-up check
- [ ] `LootEngine.php` — loot roll on quest complete
- [ ] `HPEngine.php` — HP loss for missed deadlines (checked on login)
- [ ] `CharacterController.php` — get stats, switch stone with cooldown
- [ ] `InventoryController.php` — get items
- [ ] Daily Dungeon logic — generate or fetch today's dungeon

**Frontend — HTML/CSS/JS:**
- [ ] `login.html` — register and login forms
- [ ] `index.html` — quest board layout
- [ ] `character.html` — stats, XP bar, active stone, HP
- [ ] `inventory.html` — item grid
- [ ] `api.js` — all fetch() wrappers with auth header
- [ ] `quests.js` — render quest list, filter by category/status
- [ ] `character.js` — animate XP bar fill, level-up overlay
- [ ] `stones.js` — stone picker UI, show cooldown timer
- [ ] `inventory.js` — render loot grid with rarity colours

**Test everything locally before moving on.**

---

### Phase 2 — Polish & full feature pass (still local)

Goal: everything works smoothly, no rough edges. This is the phase where it actually feels like a game.

- [ ] Level-up full-screen animation (JS)
- [ ] Loot drop animation after quest complete (JS)
- [ ] Daily Dungeon UI — countdown timer to reset, progress tracker
- [ ] Stone switcher shows cooldown ("can switch in 4 days")
- [ ] Quest filtering and sorting (by category, difficulty, deadline)
- [ ] HP warning UI when HP is low
- [ ] Missed deadline detection — run on login, deduct HP, mark quests failed
- [ ] Responsive design — make it usable on mobile screen sizes
- [ ] Dark/light theme toggle (CSS variables make this easy)
- [ ] Error handling everywhere — bad API responses shown cleanly in UI

---

### Phase 3 — Move to a live server

Goal: the app runs on a real URL, not just localhost.

**Hosting options (pick one):**
- Shared hosting (e.g. Hostinger, Namecheap) — cheapest, easy PHP/MySQL, good for learning
- VPS (e.g. DigitalOcean, Hetzner) — more control, requires setting up Apache/Nginx manually

**Steps:**
- [ ] Buy a domain (optional but nice)
- [ ] Set up hosting with PHP 8.x and MySQL support
- [ ] Switch SQLite → MySQL in `database.php` (just change the connection string)
- [ ] Upload files via FTP or SSH
- [ ] Set environment variables on the server (never upload `.env`)
- [ ] Run `schema.sql` on the live database
- [ ] Test all API routes on the live URL
- [ ] Enable HTTPS (free via Let's Encrypt)
- [ ] Set `.htaccess` to block direct access to `server/` folder from browser

---

### Phase 4 — Cross-platform apps (future)

Goal: ship a desktop app for Linux/Windows and an Android app without rewriting everything.

Since the frontend is already HTML/CSS/JS and talks to a PHP API, wrapping it is straightforward.

- [ ] **Desktop (Linux + Windows):** use Tauri — it wraps the web frontend in a native window. Lightweight, no Electron bloat.
- [ ] **Android:** use Capacitor — wraps the web app as an Android app. The PHP API stays on the server, the app just talks to it.
- [ ] Both apps point at the live server API from Phase 3.

No logic rewrite needed. The hard work is already done.

---

## Notes & Reminders

- Add `.env` to `.gitignore` on the very first commit — easy to forget
- Use `password_hash()` and `password_verify()` in PHP — never store plain passwords
- For JWT in PHP, use `firebase/php-jwt` via Composer (saves a lot of time)
- Start with SQLite locally (no MySQL setup needed when offline), switch to MySQL before Phase 3
- Stone cooldown must be checked server-side in PHP — never trust JS to enforce it
- Daily Dungeon: store today's quest IDs in `daily_dungeon` table, generate them if missing (checked on page load)
- `localStorage` is fine for caching the last-fetched character state for instant UI — but always re-fetch from PHP on load

---

*Last updated: April 2026*
