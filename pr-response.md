# PR Response Doc — CineLog Watchlist Feature

## AI Usage
<!-- Fill in at the end — how you used AI tools during this project -->

## Comment 1 — Rename
**What I did:** Renamed `save_to_watchlist()` to `add_to_watchlist()` in `services/watchlist_service.py` to follow the project's `verb_to_noun` naming convention (matching `add_to_collection()` in the collection service). Updated the one call site in `routes/watchlist/watchlist.py` — both the import statement and the invocation inside the `/add` endpoint.
**How I verified:** Ran a project-wide search for `save_to_watchlist` after the change and confirmed zero remaining references. Also imported both modules (`services.watchlist_service` and `routes.watchlist.watchlist`) to confirm the new name resolves correctly at every call site.

## Comment 2 — Deduplication
**What I did:** Added deduplication to `add_to_watchlist()` following the same three-layer pattern used by `add_to_collection()`:
1. **Service layer** — defined a new `AlreadyInWatchlistError` exception and, before inserting, query for an existing `(user_id, film_id)` `WatchlistEntry`; if one exists, raise the error instead of creating a duplicate.
2. **Model layer** — added `UniqueConstraint("user_id", "film_id", name="unique_user_film_watchlist")` to `WatchlistEntry` as a database-level backstop (mirrors `CollectionEntry`).
3. **Route layer** — the `/watchlist/<user_id>/add` endpoint now catches `AlreadyInWatchlistError` and returns HTTP 409 (also added the previously-missing `FilmNotFoundError` → 404 handling, matching the collection route).

**How I verified:** Ran a behavioral check against an in-memory DB: adding the same film twice raised `AlreadyInWatchlistError` on the second call and left exactly one entry in the table; a nonexistent film still raised `FilmNotFoundError`. The existing test suite continues to pass (4 passed).

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