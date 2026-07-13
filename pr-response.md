# PR Response Doc — CineLog Watchlist Feature

## AI Usage
<!-- Fill in at the end — how you used AI tools during this project -->

## Comment 1 — Rename
**What I did:** Renamed `save_to_watchlist()` to `add_to_watchlist()` in `services/watchlist_service.py` to match the project's `verb_to_noun` naming convention used elsewhere (e.g. `add_to_collection`). Updated the import and call site in `routes/watchlist/watchlist.py`, and updated the function's docstring to match the new name.
**How I verified:** Ran `grep -rn "save_to_watchlist" . --include="*.py"` to confirm no remaining references in source files (only a stale `__pycache__` bytecode file matched, which isn't source and regenerates automatically). Ran `pytest tests/ -v` — all existing tests passed.

## Comment 2 — Deduplication
**What I did:** Added a duplicate check to `add_to_watchlist()`, mirroring `add_to_collection()`'s pattern: after confirming the film exists, query for an existing `WatchlistEntry` with the same `user_id` and `film_id`; if found, raise a new `AlreadyInWatchlistError` instead of silently creating a duplicate entry.
**How I verified:** Ran `pytest tests/ -v` to confirm no regressions in the existing test suite.

## Comment 3 — Missing test
**What I did:** Added `tests/test_watchlist.py` with `test_add_to_watchlist_nonexistent_film_raises`, modeled directly on `test_add_to_collection_nonexistent_film_raises` in `test_collection.py`.
**How I verified:** Ran `pytest tests/test_watchlist.py -v` and confirmed it passes.

## Comment 4 — Default visibility
**My position:** Watchlists should default to `public=False` (private), not `True`.

**Reasoning:** A watchlist is different from a collection in a way that matters here: a collection shows films someone has already watched — a settled fact — while a watchlist exposes current, undecided intent. That feels more personal; I don't think users should have their "things I want to watch" list broadcast by default just because they used the feature. Someone should have to actively choose to share that, not have it exposed as a side effect of adding a film.

**Tradeoff acknowledged:** The real cost of defaulting to private is that it weakens the social/discovery angle of the feature — CineLog seems to want people bonding over shared taste, getting recommendations, or having friends weigh in on whether something's worth watching. Defaulting to public would make that happen automatically, for free, for every user. Defaulting to private means that only happens for users who opt in, which is a real loss of "free" engagement. I think that's an acceptable tradeoff because the downside of the opposite choice — someone's list being visible before they decided to share it — sits with the user personally (being judged for taste, unwanted opinions, etc.), while the downside of my choice sits with the product (slightly less viral discovery). I'd rather the product absorb that cost than the user.

## Comment 5 — Sort order
**My position:** Agree with the reviewer — sort watchlists by `date_added` descending (most recent first), matching `get_collection()`'s existing behavior. Changed `get_watchlist()`'s `.order_by(Film.title.asc())` to `.order_by(WatchlistEntry.date_added.desc())`.

**Reasoning:** Alphabetical order organizes by title, but it doesn't tell you anything about when you added something — if you're actively building a watchlist over time, you lose track of what you just added versus what's been sitting there a while, since a new addition can land anywhere in the list depending on its title. Date-added order keeps the most recent addition visible at the top, which matches how people actually use a "want to watch" list.

**Engagement with reviewer's point:** I agree with the reviewer's stated reasoning — "most users want to see what they added recently" matches my own instinct about how this feature gets used. Beyond agreeing on the merits, there's a consistency benefit too: `get_collection()` already sorts by `date_added.desc()`, so making watchlist behave the same way means users don't have to relearn how sorting works when moving between the two features — one consistent mental model across the app.

## Comment 6 — Rebase
**What conflicted:** `.gitignore` conflicted trivially (both branches had independently added one). More significantly, `models.py`'s merge during the rebase silently dropped the `WatchlistEntry` class entirely rather than properly integrating it alongside main's UUID refactor — this wasn't flagged as an explicit conflict, so it went unnoticed until I ran the test suite and got an `ImportError`.
**How I resolved it:** Resolved the `.gitignore` conflict by keeping the union of both sides' ignore patterns. Re-added `WatchlistEntry` to `models.py`, updating `film_id` to `db.String(36)` to match the refactored `Film.id` (previously `db.Integer`). Also updated the docstring in `routes/watchlist/watchlist.py` (`film_id` request body type) and the fake film ID in `tests/test_watchlist.py` from an integer literal to a UUID-format string, since the column type had changed.
**How I verified no conflict remains:** Ran `pytest tests/ -v` — all 5 tests pass. Confirmed no lingering conflict markers in source files with `grep -rn "<<<<<<<\|=======\|>>>>>>>" --include="*.py" --include="*.md" . --exclude-dir=.venv`.

## PR Description
<!-- Written at the end — feature overview, design decisions, manual testing steps -->