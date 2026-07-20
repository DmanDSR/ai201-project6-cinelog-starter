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
**What I did:** Created `tests/test_watchlist.py`, mirroring the fixture and assertion structure of `test_collection.py`. Added `test_add_to_watchlist_nonexistent_film_raises` — the direct equivalent of `test_add_to_collection_nonexistent_film_raises` — which asserts that calling `add_to_watchlist()` with a film_id that isn't in the database raises `FilmNotFoundError` (rather than a raw DB integrity error). Reused the same `app`, `sample_user`, and `sample_film` fixtures (in-memory SQLite, created/dropped per test). One adaptation: film IDs are integers on this branch, so the fake ID is `99999` instead of the collection test's UUID string.
**How I verified:** Ran `pytest tests/test_watchlist.py -v` — 1 passed. Full suite also green (5 passed).

## Comment 4 — Default visibility
**My position:** Keep `public=True` as the default for new watchlist entries. This is an intentional decision, not an inherited default.

**Reasoning:** CineLog is a social film-tracking network, and the watchlist is fundamentally a *discovery and social* surface — it answers "what does this person plan to watch?" The behavior I'm optimizing for is frictionless participation in that social graph: a new user who saves a film should immediately contribute to their friends' discovery feeds without first having to hunt for a privacy toggle. Public-by-default is what makes the network effect work — the product gets more valuable as more watchlist activity is visible, recommendations improve, and users can see and discuss what people they follow want to watch. Requiring an opt-in to public would leave most lists private (defaults are sticky), which would quietly gut the social feature this PR exists to build. The `public` field being per-entry also means users retain granular control — the default is a starting point, not a lock-in.

**Tradeoff acknowledged:** The real cost is privacy: a public default means a user who *assumes* their watchlist is private can unintentionally expose their viewing intentions (which can be sensitive — health, religion, sexuality can all be inferred from film choices). The more conservative alternative, `public=False` (privacy-by-default), aligns with data-minimization principles and GDPR's "privacy by design," and eliminates the accidental-exposure risk entirely; its cost is a colder start for the social features and an extra step for the majority of users who *do* want to share. I'm accepting the privacy tradeoff, but it's contingent on three mitigations that should ship alongside this default: (1) the visibility state must be clearly shown in the UI at creation time so it's never a surprise, (2) an easy per-entry public/private toggle, and (3) a first-run prompt the first time a user creates a watchlist so the default is a conscious choice. If we can't commit to at least (1) and (2), I'd switch the default to `public=False`.

## Comment 5 — Sort order
**My position:**
**Reasoning:**
**Engagement with reviewer's point:**

## Comment 6 — Rebase
**What conflicted:** Rebased `feature/watchlist` onto `main` (`git rebase main`), replaying 6 commits over main's int→UUID migration. The single content conflict was in `models.py`: main's UUID migration had removed the `WatchlistEntry` class, while my dedup commit tried to edit it. `app.py` (watchlist blueprint registration) and `services/collection_service.py` (the `db.session.get` switch) both replayed cleanly.

**How I resolved it:** Re-added the `WatchlistEntry` class on top of main's UUID models, changing `film_id` from `db.Integer` to `db.String(36)` so its foreign key matches the migrated `Film.id` (UUID). Kept the dedup `UniqueConstraint` from Comment 2. Then updated the remaining integer-ID assumptions "accordingly": the `add_to_watchlist` docstring (`film_id (int)` → `film_id (str): UUID`), the route's request-body doc (`<int>` → `"<uuid>"`), and the test's fake ID (`99999` → a UUID string, matching `test_collection.py`).

**How I verified no conflict remains:** Searched the tree for conflict markers (none). App boots and registers the watchlist blueprint (`/watchlist/<user_id>`, `/watchlist/<user_id>/add`). Behavioral check with a real UUID film confirmed `add_to_watchlist` persists `entry.film_id == film.id`, and dedup still raises `AlreadyInWatchlistError`. Full suite green (`pytest tests/ -v` → 5 passed). (Separately, verification surfaced a pre-existing bug — `get_watchlist()` references a `WatchlistEntry.film` relationship that was never defined — which I'll fix under Comment 5, since that comment concerns `get_watchlist` ordering.)

## PR Description

Adds a watchlist feature so users can save films they want to watch. Includes a new `WatchlistEntry` model, service functions (`add_to_watchlist`, `get_watchlist`), REST endpoints, and deduplication (service check + DB unique constraint).

**Default visibility decision:** New watchlist entries default to `public=True`. This is intentional: CineLog is a social film-tracking network and the watchlist is a discovery surface, so a public default lets users participate in the social graph without friction and powers friend-based discovery. The tradeoff is privacy — a public default risks users unintentionally exposing their viewing intentions — which we accept only alongside a clearly-visible visibility state in the UI, an easy per-entry public/private toggle, and a first-run prompt. (Full reasoning in "Comment 4" above.)