# PR Response Doc — CineLog Watchlist Feature

## AI Usage
<!-- Fill in at the end — how you used AI tools during this project -->

## Comment 1 — Rename
**What I did:** Renamed `save_to_watchlist()` to `add_to_watchlist()` in `services/watchlist_service.py` to follow the project's `verb_to_noun` naming convention (matching `add_to_collection()` in the collection service). Updated the one call site in `routes/watchlist/watchlist.py` — both the import statement and the invocation inside the `/add` endpoint.
**How I verified:** Ran a project-wide search for `save_to_watchlist` after the change and confirmed zero remaining references. Also imported both modules (`services.watchlist_service` and `routes.watchlist.watchlist`) to confirm the new name resolves correctly at every call site.

## Comment 2 — Deduplication
**What I did:**
**How I verified:**

## Comment 3 — Missing test
**What I did:**
**How I verified:**

## Comment 4 — Default visibility
**My position:**
**Reasoning:**
**Tradeoff acknowledged:**

## Comment 5 — Sort order
**My position:**
**Reasoning:**
**Engagement with reviewer's point:**

## Comment 6 — Rebase
**What conflicted:**
**How I resolved it:**
**How I verified no conflict remains:**

## PR Description