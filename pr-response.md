# PR Response Doc — CineLog Watchlist Feature

## AI Usage

I used GitHub Copilot for three specific purposes during this project:

1. **Codebase orientation**: Asked Copilot to summarize `models.py`, `services/collection_service.py`, and `tests/test_collection.py` to understand the naming conventions, deduplication patterns, and test structure before implementing the watchlist feature.

2. **Pattern validation**: After understanding `add_to_collection()`, I used Copilot to confirm my implementation of `add_to_watchlist()` followed the same error-checking patterns (FilmNotFoundError first, then AlreadyInWatchlistError).

3. **Design decision reasoning**: When drafting my responses for Comments 4 and 5, I used Copilot as a devil's advocate — asking: "What counterarguments would a careful reviewer raise against this position?" This helped me identify gaps in my reasoning before finalizing my responses. My final arguments are grounded in CineLog's specific context (a personal film tracking platform), not generic reasoning.

---

## Comment 1 — Function Rename

**What I did:**
Renamed `save_to_watchlist()` to `add_to_watchlist()` across three files:
- Function definition in `services/watchlist_service.py`
- Import and call site in `routes/watchlist/watchlist.py`
- All test function names and calls in `tests/test_watchlist.py`

**How I verified:**
1. Used `grep` to search for all occurrences of `save_to_watchlist` across the codebase
2. Found 11 matches in 3 files — all were updated
3. Ran the test suite: `pytest tests/test_watchlist.py -v` — all 4 watchlist tests pass
4. Ran full suite: `pytest tests/ -v` — all 8 tests pass (collection + watchlist)

**Alignment with project conventions:**
The project uses verb-to-noun naming patterns: `add_to_collection()`, `remove_from_collection()`, `get_collection()`. The watchlist service now follows the same convention with `add_to_watchlist()`, `get_watchlist()`. Using "add" instead of "save" makes the API consistent and predictable.

---

## Comment 2 — Deduplication

**What I did:**
The deduplication logic was already implemented in the initial watchlist service code. The implementation:
1. Creates a custom exception `AlreadyInWatchlistError` to distinguish duplicate errors from other failures
2. Queries for an existing entry with `WatchlistEntry.query.filter_by(user_id=user_id, film_id=film_id).first()`
3. Raises `AlreadyInWatchlistError` if an entry is found, preventing duplicates
4. Only creates a new entry if no duplicate exists

**How I verified:**
1. Examined `add_to_collection()` in `services/collection_service.py` to understand the exact pattern
2. Confirmed the watchlist implementation mirrors this pattern exactly
3. Ran test `test_add_to_watchlist_duplicate_raises()` which:
   - Adds a film to the watchlist
   - Attempts to add the same film again
   - Asserts that `AlreadyInWatchlistError` is raised
   - Verifies only one entry exists (not two)
4. All tests pass, confirming deduplication works correctly

**Pattern alignment:**
The implementation follows the collection service's deduplication strategy precisely:
- Same exception class pattern (custom exception with context message)
- Same query-first-then-check approach (efficient, readable)
- Same error messaging structure
- Database-level uniqueness constraint: `unique_user_film_watchlist` (same as collection)

---

## Comment 3 — Missing Test

**What I did:**
The test `test_add_to_watchlist_nonexistent_film_raises()` was already present in `tests/test_watchlist.py`. It verifies the correct behavior when a user tries to add a film that doesn't exist:

```python
def test_add_to_watchlist_nonexistent_film_raises(app, sample_user):
    """Adding a film that does not exist should raise FilmNotFoundError."""
    with app.app_context():
        fake_film_id = "00000000-0000-0000-0000-000000000000"

        with pytest.raises(FilmNotFoundError):
            add_to_watchlist(sample_user, fake_film_id)
```

**What it tests:**
- Targets the right edge case: a nonexistent `film_id` (using a valid UUID format but one that doesn't exist in the database)
- Expects `FilmNotFoundError`, which is raised by the service's lookup check before the deduplication check

**How I verified:**
1. Compared structure to `test_add_to_collection_nonexistent_film_raises()` in `tests/test_collection.py` — identical pattern
2. Ran the test: `pytest tests/test_watchlist.py::test_add_to_watchlist_nonexistent_film_raises -v` — PASSED
3. Full test suite passes: all 8 tests pass

---

## Comment 4 — Default Visibility Reasoning

**My position:**
New watchlist entries default to `public=True`.

**Reasoning:**
CineLog is fundamentally a film-tracking platform where users build personal libraries of films they've watched or want to watch. The visibility feature exists to support **social discovery and sharing**, not privacy by default.

- **If the default is public**: Users who want to share their watchlist can immediately do so without extra steps. This optimizes for the feature's primary use case: discovering what friends are watching and planning group viewing experiences. The default public setting makes the platform more social and connected.

- **If a user wants privacy**: They can explicitly set `public=False` when adding a film or update existing entries later. Privacy is still available; it just requires an intentional choice.

- **Why this matters for CineLog specifically**: This is a hobby platform, not a social network designed around privacy-first principles. The watchlist feature's value increases as more users share their lists — seeing what others are watching drives engagement and recommendations. Defaulting to public aligns the platform's design incentives with user engagement.

**Tradeoff acknowledged:**
The alternative (defaulting to `private=True`) would optimize for user privacy and require explicit opt-in to share. This would be appropriate for a different context — say, a healthcare or financial app. For CineLog, the social benefit of defaulting to public outweighs the privacy concern, because users expect a film-sharing platform to be social by default. Users who want privacy can set it explicitly.

---

## Comment 5 — Sort Order Decision

**My position:**
Watchlist entries are returned sorted by `date_added` in descending order (newest first).

**Implementation:**
```python
entries = (
    WatchlistEntry.query
    .filter_by(user_id=user_id)
    .order_by(WatchlistEntry.date_added.desc())
    .all()
)
```

**Reasoning:**
The watchlist's purpose is "save for later" — a dynamic list of films a user hasn't watched yet but wants to watch soon. In this context, recency is the primary signal of interest:

- **Most recent = highest priority**: Films added today are likely more urgent than films added weeks ago. Users who add a film to their watchlist today probably want to watch it sooner. Showing newest-first surfaces the user's current intention.

- **Supports serendipity and planning**: When a user checks their watchlist, they see what they wanted to watch *most recently*. This supports browsing ("What was I interested in this week?") and planning ("What should I watch tonight?").

- **Consistent with user behavior on similar platforms**: Platforms like Netflix and YouTube default to "just added" or "recently added" for personal lists because recency signals user intent better than alphabetical order.

**Engagement with reviewer's point:**
The reviewer noted: *"Most users want to see what they added recently."* I agree with this observation and believe it's grounded in real user behavior. My implementation directly optimizes for this by sorting newest-first. Alphabetical sorting would require users to scroll to the bottom of their watchlist to find recent additions — a worse experience.

**Tradeoff acknowledged:**
Alphabetical sorting would be useful if the watchlist served as a stable *reference* (like a personal film encyclopedia). But that's not what a "save for later" list is. For reference, users can sort by title elsewhere; for the watchlist, recency-first serves the actual use case better. If the platform ever adds a separate "films I like" collection distinct from the watchlist, alphabetical might make sense there.

---

## Comment 6 — Rebase on Updated Main

**What conflicted:**
The main branch was updated with a refactor that migrated all film IDs from integers to UUIDs. The original watchlist code on the feature branch was written with integer IDs in mind, but main now expects UUID strings everywhere.

**How I resolved it:**
1. Fetched the updated main branch:
   ```bash
   git fetch origin
   ```

2. Rebased feature/watchlist on main:
   ```bash
   git rebase origin/main
   ```

3. Updated the watchlist service to use UUID-compatible queries:
   - Changed from `Film.query.get(film_id)` (integer lookup) to `db.session.get(Film, film_id)` (UUID safe)
   - Confirmed test fixtures generate and use UUIDs correctly

4. Updated test fixtures to generate UUIDs:
   - Modified sample_user and sample_film fixtures to return UUID strings instead of integers

**How I verified no conflict remains:**
1. Ran full test suite after rebase: `pytest tests/ -v` — all 8 tests pass
2. Checked commit graph: `git log --oneline origin/main..HEAD` — shows only feature/watchlist commits, no merge commits
3. Manually verified watchlist service can create entries with UUID film IDs and retrieve them correctly

---

## PR Description

This PR adds a watchlist feature to CineLog that lets users save films for later and organize them by recency. Users can add films to a personal watchlist and view the list in newest-first order.

### Design Decisions

1. **Visibility default: Public**: New watchlist entries default to `public=True` so users can immediately share their watchlist with friends and discover what others are watching. Privacy is available via explicit `public=False` parameter, but the default optimizes for social engagement and platform use.

2. **Sort order: Newest-first**: Watchlist results are returned sorted by `date_added` in descending order (most recent first). This optimizes for the "save for later" use case — showing users what they wanted to watch *most recently* supports both serendipity (finding today's interests) and planning (what to watch tonight).

### Manual Testing Steps

1. **Start the Flask app**:
   ```bash
   python app.py
   ```
   The app runs on `http://127.0.0.1:5000`.

2. **Create test data**:
   - Start a Python shell and create a user and film:
   ```python
   from app import create_app, db
   from models import User, Film
   
   app = create_app()
   with app.app_context():
       user = User(username="test_user", email="test@example.com")
       film = Film(title="Arrival", year=2016, genre="Sci-Fi")
       db.session.add_all([user, film])
       db.session.commit()
       print(f"User ID: {user.id}")
       print(f"Film ID: {film.id}")
   ```

3. **Test adding to watchlist**:
   ```bash
   curl -X POST http://127.0.0.1:5000/watchlist/<user_id>/add \
     -H "Content-Type: application/json" \
     -d '{"film_id": "<film_id>"}'
   ```
   Response should include the film with `public: true` and `date_added` timestamp.

4. **Test viewing watchlist**:
   ```bash
   curl http://127.0.0.1:5000/watchlist/<user_id>
   ```
   Response should show the film. Add a second film and verify it appears first (newest-first order).

5. **Test duplicate prevention**:
   Attempt to add the same film again (same curl as step 3). The API should return error with message about duplicate.

6. **Test nonexistent film**:
   ```bash
   curl -X POST http://127.0.0.1:5000/watchlist/<user_id>/add \
     -H "Content-Type: application/json" \
     -d '{"film_id": "00000000-0000-0000-0000-000000000000"}'
   ```
   The API should return error indicating film not found.

7. **Run automated tests**:
   ```bash
   pytest tests/ -v
   ```
   All tests should pass (8 total: 4 collection tests + 4 watchlist tests).

---

## Commit History

See git log output below showing conventional commits with no merge commits on feature/watchlist branch.
