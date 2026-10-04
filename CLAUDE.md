# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**GoTabOut (TabOut)** — Chrome Manifest V3 extension that replaces the new tab page with a dashboard of open tabs grouped by domain. 100% local, no server, no build step. Based on [zarazhangrui/tab-out](https://github.com/zarazhangrui/tab-out).

## Development

No build system. No npm. No tests. Edit files in `extension/`, reload in `chrome://extensions`.

**Test changes:** open `chrome://extensions`, click the reload icon on TabOut, then open a new tab.

**Keyboard shortcut:** `Cmd+Shift+K` (Mac) / `Ctrl+Shift+K` (Windows) opens or focuses the dashboard.

## Architecture

All runtime code lives in `extension/`.

### Entry Points

- **`index.html`** — Dashboard UI. Loads `style.css`, `fonts.css`, `theme-init.js` (sets `data-theme` before paint to avoid flash), then `app.js`.
- **`background.js`** — Service worker. Three jobs:
  1. Toolbar click / shortcut → open or focus dashboard tab
  2. `chrome.tabs.onUpdated` → redirect `chrome://newtab/` to dashboard when `settings.defaultPage` is true
  3. Badge updater — counts "real" tabs, color-codes (green ≤10, amber ≤20, red >20)

### Main Logic (`app.js`)

Single ~1700-line file. No modules, no bundler. Key sections (top to bottom):

1. **Tab management** — `fetchOpenTabs()`, `closeTabsWhere()`, `closeTabsByUrls()`, `closeTabsExact()`, `closeDuplicateTabs()`, `focusTab()`. Uses hostname matching for bulk close, exact URL match for landing pages.
2. **Saved-for-later (deferred)** — `chrome.storage.local` key `deferred`. Items have `completed`/`dismissed` flags.
3. **UI helpers** — confetti particles, swoosh sound (Web Audio synthesis, no audio files), card-close animations, toast notifications.
4. **Domain cleanup** — `FRIENDLY_DOMAINS` map (e.g. `github.com` → `GitHub`), `cleanTitle()` strips domain suffixes from page titles, `smartTitle()` generates readable titles for known URL patterns (tweet, PR, issue, Reddit post).
5. **Settings** — `chrome.storage.local` key `settings`. Fields: `logoUrl`, `defaultPage`, `showTabList`, `tabListItems`, `backgroundImage`. `DEFAULT_SETTINGS` is the source of truth.
6. **Nav list** — `renderTabList()`. Items stored as `{url, title}`. Supports custom names via `"url name"` format in textarea. Drag-and-drop reorder via HTML5 drag events (chips are `<div>` not `<a>` to avoid native link-drag interference). Close button on each chip removes from nav list.
7. **Tab grouping** — `renderStaticDashboard()` builds `domainGroups` array. Landing pages (Gmail inbox, X home, etc.) get their own `__landing-pages__` group so closing them doesn't nuke content tabs on same hostname. `LOCAL_LANDING_PAGE_PATTERNS` and `LOCAL_CUSTOM_GROUPS` can be injected via `config.local.js` (not in repo).
8. **Event delegation** — single `document.addEventListener('click')` at line ~1165 handles all `[data-action]` clicks. Actions: `focus-tab`, `close-single-tab`, `defer-single-tab`, `add-to-nav-list`, `remove-nav-item`, `check-deferred`, `edit-deferred-title`, `dismiss-deferred`, `close-domain-tabs`, `dedup-keep-one`, `close-all-open-tabs`, `open-all-deferred`, `clear-all-deferred`, `close-tabout-dupes`, `expand-chips`.
9. **Live refresh** — `chrome.tabs.onCreated/onRemoved/onUpdated/onActivated` → `scheduleRefresh()` → debounced `renderDashboard()`. 300ms normal, 800ms during manual close animations.

### Storage Schema (`chrome.storage.local`)

- `settings` — `{logoUrl, defaultPage, showTabList, tabListItems, backgroundImage}`
- `deferred` — `[{id, url, title, favIconUrl, savedAt, completed, completedAt, dismissed}]`

### Styling (`style.css`)

CSS custom properties for theming. `data-theme="dark"` on `<html>` for dark mode. Warm paper aesthetic (`--paper`, `--ink`, `--warm-gray`, `--accent-amber`, `--accent-sage`). `body::before` adds subtle SVG noise texture. Background image support via `#backgroundLayer` fixed element at z-index -1.

### Theming (`theme-init.js`)

Runs in `<head>` before paint. Reads `localStorage.theme`, falls back to `prefers-color-scheme`. Sets `data-theme` attribute immediately to avoid flash of wrong theme.

## Adding Features

- **New action button:** add `data-action="your-action"` attribute, handle in the click delegation block (~line 1165).
- **New setting:** add to `DEFAULT_SETTINGS`, add UI to `index.html` settings modal, wire up in `initSettings()` (load + commit + apply).
- **New icon:** add SVG path to `ICONS` object using the `svg(strokeWidth, path)` helper. Heroicons outline style.
- **Landing page pattern:** add to `LANDING_PAGE_PATTERNS` array in `renderStaticDashboard()`, or inject via `config.local.js`.
