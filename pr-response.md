# PR Response Doc — CineLog Watchlist Feature

## AI Usage

I used AI (Claude) throughout this project in a few distinct ways:

**Codebase orientation:** Before touching any review comments, I had the AI walk me through `models.py`, `services/collection_service.py`, and `tests/test_collection.py` to understand the existing naming conventions, deduplication pattern, and test fixture structure. This made the six review comments much easier to parse — e.g. understanding why the reviewer wanted `save_to_watchlist` renamed (`verb_to_noun` convention already established by `add_to_collection`).

**Hygiene and verification:** I used AI to help verify things like confirming no remaining references to `save_to_watchlist` existed anywhere in source (`grep --include="*.py"`), and to help debug environment issues (a broken venv after re-forking, a Vim editor I didn't know how to exit, and eventually a serious rebase problem).

**Catching a rebase issue:** After rebasing on `main`, my tests passed initially, but the AI pushed me to specifically verify that `WatchlistEntry` had actually survived the merge rather than assuming success from Git's "Successfully rebased" message. This caught a real bug — the `.gitignore` conflict resolution had silently dropped the entire `WatchlistEntry` class from `models.py`, which only surfaced as an `ImportError` when running the test suite. I would not have caught this without being prompted to verify rather than trust the rebase output.

**Comments 2, 4, and 5 :** The AI would only ask me guiding questions (e.g. "would you personally want your watchlist public or private by default?") and helped me polish the wording of my own answers afterward, rather than generating the reasoning itself. My final positions and arguments for both comments are my own reasoning, not AI-generated.

**Where I did accept AI-drafted content:** I did ask the AI to write `tests/test_watchlist.py` directly (Comment 3), since that wasn't flagged with the same restriction as Comment 2, and I asked it to compile/format the mechanical sections of this document (Comments 1, 2, 3, 6, and this section) based on facts I'd already reported to it.

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

**What this PR does:**
Adds a watchlist feature to CineLog, allowing users to save films they intend to watch (as distinct from the existing collection feature, which tracks films already watched). Includes a `WatchlistEntry` model, `add_to_watchlist()` / `get_watchlist()` service functions, and REST endpoints (`GET /watchlist/<user_id>`, `POST /watchlist/<user_id>/add`).

**Changes made in response to review:**
- Renamed `save_to_watchlist()` to `add_to_watchlist()` to match the project's `verb_to_noun` naming convention (Comment 1).
- Added deduplication logic to `add_to_watchlist()`, raising `AlreadyInWatchlistError` if a film is already on the user's watchlist, following the same pattern as `add_to_collection()` (Comment 2).
- Added `tests/test_watchlist.py` covering the nonexistent-film-id case (Comment 3).
- Rebased `feature/watchlist` onto `main` to pick up the integer-to-UUID film ID refactor, and fixed a `WatchlistEntry` model regression that the rebase's `.gitignore` conflict resolution silently introduced (Comment 6).

**Design decisions:**
- **Default visibility:** Watchlists default to `public=False` (private). A watchlist reveals current, undecided viewing intent rather than settled history, which feels more personal than a collection — users should opt in to sharing rather than have it exposed by default. This trades off some of the app's built-in social/discovery value, which I think is an acceptable cost since the alternative risk (unwanted exposure) sits with the user rather than the product.
- **Sort order:** Watchlists are sorted by `date_added` descending (most recent first), matching `get_collection()`'s existing behavior. This keeps the most recently added film visible at the top, and gives users one consistent sorting mental model across both features.

**How to manually test:**
```bash
# Start the app
python app.py

# Add a film to a user's watchlist
curl -X POST http://127.0.0.1:5000/watchlist/<user_id>/add \
  -H "Content-Type: application/json" \
  -d '{"film_id": "<existing-film-uuid>"}'

# View the watchlist (should show newest addition first)
curl http://127.0.0.1:5000/watchlist/<user_id>

# Attempt to add the same film again — should return an error (AlreadyInWatchlistError)
curl -X POST http://127.0.0.1:5000/watchlist/<user_id>/add \
  -H "Content-Type: application/json" \
  -d '{"film_id": "<same-film-uuid>"}'

# Attempt to add a nonexistent film — should return FilmNotFoundError
curl -X POST http://127.0.0.1:5000/watchlist/<user_id>/add \
  -H "Content-Type: application/json" \
  -d '{"film_id": "00000000-0000-0000-0000-000000000000"}'
```

Run the automated test suite as well:
```bash
pytest tests/ -v
```

<img width="740" height="201" alt="Screenshot 2026-07-13 at 22 44 39" src="https://github.com/user-attachments/assets/bbe9f5b0-d477-4854-8d1d-540660ad7e09" />
