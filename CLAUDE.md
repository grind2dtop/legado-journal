# CLAUDE.md — Legado Journal

This file gives AI assistants the context they need to work effectively on this codebase.

---

## Project Overview

**Legado** is a personal living journal for Roldan, a Puerto Rican Navy veteran and entrepreneur. It uses Claude AI (via a Cloudflare Worker proxy) to transform raw voice transcriptions or stream-of-consciousness thoughts into polished, autobiography-style journal entries. Entries are stored in Google Firebase Firestore and browsable by five "life pillars."

This is a **zero-dependency, single-file web application**. Everything — HTML, CSS, JavaScript — lives in `index.html`. There is no build system, no bundler, no package.json, no test framework.

---

## Repository Layout

```
legado-journal/
├── index.html     # The entire application (471 lines)
└── README.md      # Brief project description
```

That's it. Do not introduce build tooling, frameworks, or additional files unless explicitly requested.

---

## Tech Stack

| Layer | Technology |
|---|---|
| Frontend | Vanilla HTML5, CSS3, ES2020+ JavaScript (no framework) |
| Database | Google Firebase Firestore (via REST API) |
| AI | Anthropic Claude API (`claude-sonnet-4-20250514`) |
| Proxy | Cloudflare Worker (`legado-proxy.roldan-crespo-ai.workers.dev`) |
| Auth storage | Browser `localStorage` |

---

## Architecture: Single-File SPA

The app is a hand-rolled single-page application with four screens and a bottom navigation bar. All screens are `<div>` elements that are hidden/shown by toggling the `.active` CSS class via the `show(name)` function.

### Screens

| Screen ID | Purpose |
|---|---|
| `screen-setup` | One-time Anthropic API key entry (shown on first load) |
| `screen-capture` | Input raw thoughts to be transformed by AI |
| `screen-journal` | Browse all entries; filter by pillar |
| `screen-entry` | Read a single entry in full; delete it |

### Navigation flow

```
First visit → setup → capture ⇄ journal → entry → (back to journal or capture)
```

After setup the bottom nav (`#bottom-nav`) becomes visible. `show('journal')` always triggers a fresh `loadEntries()` from Firestore.

---

## Key Constants (top of `<script>`)

```js
FB_PROJECT  // "legado-journal" — Firebase project ID
FB_API_KEY  // Firebase Web API key (safe to be public; Firestore rules enforce access)
FB_BASE     // Firestore REST base URL for the `entries` collection
PILLARS     // Array of 5 pillar objects {id, label, icon, color}
PROMPT      // System prompt sent to Claude — defines Penelope's persona and output schema
```

### The Five Pillars

| ID | Label | Icon | Hex color |
|---|---|---|---|
| 1 | Self-Care & Growth | 🙏 | `#7C3AED` |
| 2 | Fatherhood | 👨‍👧‍👦 | `#059669` |
| 3 | Swing Trading | 📈 | `#0284C7` |
| 4 | Keredan | 🏠 | `#D97706` |
| 5 | AI & Automation | 🤖 | `#DB2777` |

Adding, removing, or renaming pillars requires updating only the `PILLARS` array; the rest of the UI renders dynamically from it.

---

## Global State

```js
let entries = [];        // In-memory cache of all Firestore documents
let activeFilter = null; // Pillar ID (1–5) or null for "All"
```

State is never persisted to `localStorage` beyond the API key (`legado_akey`).

---

## Firestore Data Schema

Each entry document in the `entries` collection:

```json
{
  "date":    "2026-03-30T12:00:00.000Z",  // ISO 8601 string
  "title":   "Short evocative title",
  "entry":   "Full autobiography-style journal entry",
  "raw":     "Original raw user input",
  "mood":    "grateful",
  "pillars": [1, 3]                        // Array of pillar IDs (integers)
}
```

Firestore documents use the typed REST format (`{ stringValue: "..." }`, `{ integerValue: N }`). Conversion happens in `fbGet()` and `fbAdd()`.

---

## Claude AI Integration

- **Model:** `claude-sonnet-4-20250514`
- **Endpoint:** Routed through a Cloudflare Worker proxy at `https://legado-proxy.roldan-crespo-ai.workers.dev/v1/messages` (required to avoid CORS; adds the `anthropic-dangerous-request-header: true` forwarding)
- **API key:** User-supplied, stored in `localStorage` as `legado_akey`, sent as `x-api-key` header
- **Output contract:** Claude must return **strict JSON only** (no markdown fences):
  ```json
  {"title":"...","entry":"...","pillars":[1,2],"mood":"..."}
  ```
  The `transformThought()` function strips any accidental backtick fences before `JSON.parse()`.

The AI persona is named **Penelope**. The `PROMPT` constant defines her deeply — do not change it without understanding Roldan's personal context.

---

## Core Functions Reference

| Function | Description |
|---|---|
| `saveSetup()` | Validates and saves the Anthropic API key to localStorage |
| `resetSetup()` | Clears the API key, returns to setup screen |
| `transformThought()` | Sends raw input to Claude, saves result to Firestore, shows new entry |
| `loadEntries()` | Fetches all entries from Firestore, calls `renderJournal()` |
| `renderJournal()` | Re-renders the journal list and pillar filter buttons |
| `showEntry(id)` | Renders a single entry's detail view |
| `deleteEntry(id)` | Confirms, deletes from Firestore, updates in-memory array |
| `setFilter(pid)` | Sets `activeFilter` and re-renders the journal |
| `exportEntries()` | Downloads all entries as a `legado-backup-YYYY-MM-DD.json` file |
| `show(name)` | SPA router — toggles screen visibility and bottom nav active state |
| `fbGet()` | Firestore REST: fetch all entries, sorted newest-first |
| `fbAdd(data)` | Firestore REST: create new entry, returns generated document ID |
| `fbDelete(id)` | Firestore REST: delete entry by document ID |

---

## Naming Conventions

- **Variables & functions:** `camelCase`
- **Constants:** `UPPER_SNAKE_CASE`
- **HTML IDs:** `kebab-case` with semantic prefix (`screen-`, `nav-`, `-btn`, `-input`, `-error`)
- **CSS classes:** `kebab-case` descriptive names
- **Event handlers:** Inline `onclick` attributes calling global functions directly (no `addEventListener`)

---

## CSS Conventions

- All styles are inline in the `<style>` block in `<head>` — no external stylesheets.
- Design language: dark glassmorphism (deep purple/indigo gradient background, frosted-glass cards).
- Colors: primary gradient `#7C3AED → #DB2777`; muted text `#a89fc0`; subtle text `#6b5f8a`.
- Font: `Georgia, serif` throughout.
- Responsive via `max-width: 680px; margin: 0 auto` on `.screen`.
- Fixed bottom nav with `backdrop-filter: blur(10px)`.

---

## Development Workflow

There is no build step. To develop:

1. Edit `index.html` directly.
2. Open the file in a browser (or serve with any static server, e.g. `python3 -m http.server`).
3. Refresh to see changes.

To deploy: upload `index.html` to any static host (GitHub Pages, Netlify, Vercel, S3, Cloudflare Pages, etc.).

---

## Environment / Secrets

| Secret | Where stored | Notes |
|---|---|---|
| Anthropic API key | Browser `localStorage` (`legado_akey`) | User-provided at setup; never committed |
| Firebase API key | Hardcoded in `index.html` | Intentionally public; Firestore security rules control access |
| Cloudflare Worker URL | Hardcoded in `transformThought()` | Proxy owned by Roldan |

There is no `.env` file. Do not add one.

---

## Known Issues / Active Bugs

- **`exportEntries()` (line 265):** Uses `Entries.length` (capital E) which is `undefined`. Should be `entries.length`. The export button will silently fail with an alert even when entries exist. Fix: change `Entries.length` → `entries.length`.

---

## What NOT to Do

- Do not add a build system, npm, webpack, Vite, or any bundler unless explicitly requested.
- Do not introduce React, Vue, or any other framework.
- Do not split the single file into multiple files.
- Do not add TypeScript.
- Do not add a `.env` file or externalize configuration.
- Do not add error handling for impossible paths (trust Firestore's API contract).
- Do not over-abstract: three similar lines are fine; premature helpers are not.
- Do not modify the `PROMPT` constant without understanding Roldan's personal context — it defines the AI's persona and directly affects every future journal entry.

---

## Git

- Main branch: `main`
- All development happens on feature branches; merge to `main` when complete.
- Commit messages are imperative, present-tense, single-line (e.g., `Fix variable name in exportEntries function`).
- Only `index.html` has ever been committed (besides this file).
- GPG/SSH commit signing is configured.
