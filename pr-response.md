# PR Response Doc — CineLog Watchlist Feature

## AI Usage
I used Claude Code throughout this project, mostly for orientation and double-checking my work.

Before touching any code, I had it read through models.py, services/collection_service.py, and tests/test_collection.py and summarize the naming conventions, the dedup pattern in add_to_collection, and the fixture structure. This let me understand the conventions and patterns for the project so that I could replicate them.
For the dedup logic specifically, I had it walk me through add_to_collection line by line so I understood the check order (film exists, then dedup, then insert) before writing add_to_watchlist myself.
I used it to run a quick Flask test-client script (two back-to-back adds, plus a bad film_id) to actually confirm the dedup logic returns 201/409/404, instead of just trusting the code by eye.
 For Comments 4 and 5, after I drafted my positions I asked what a skeptical reviewer would push back on. For the visibility default, the pushback was that collection having no privacy toggle at all might be a gap worth fixing, not a precedent worth copying so I made sure to acknowledge the discoverability risk head-on instead of leaning on consistency as if that settles it. For sort order, the pushback was that oldest-first has its own failure mode (a truly abandoned entry could sit at the top forever), so I stopped short of claiming oldest-first is a universal improvement and framed it as the better fit for this specific list.

## Comment 1 — Rename
**What I did:** Renamed save_to_watchlist to add_to_watchlist in the watchlist service so it follows the verb_to_noun convention CONTRIBUTING.md documents and add_to_collection already uses. Updated the one call site in the watchlist route.

**How I verified:** Grepped the repo for the old name before and after, one hit in the service, one in the route, zero left over. Ran the full test suite to make sure nothing broke.

## Comment 2 — Deduplication
**What I did:** Copied the dedup pattern straight from add_to_collection. Added an AlreadyInWatchlistError, and before inserting a new entry I now check whether one already exists for that user and film. I also added a NotInWatchlistError while I was in there, so the exception vocabulary would already be in place for remove_from_watchlist later. Since the collection route catches these service errors and turns them into status codes, I did the same for the watchlist route.

**How I verified:** Traced through add_to_collection's check order by hand to make sure I matched it. Ran the full suite to confirm the collection tests were untouched, then manually hit the endpoint twice with the same film — first call 201, second call 409, no duplicate row.

## Comment 3 — Missing test
**What I did:** Created tests/test_watchlist.py, copying the fixture setup from test_collection.py. Used the nonexistent-film test for collection as the direct template for the watchlist version. I also added happy-path and duplicate tests, since CONTRIBUTING.md asks for all three whenever a service function is added, and neither existed yet for watchlist.

**How I verified:** Ran the new file on its own, then the full suite, to make sure the two test files don't step on each other (separate fixtures, separate in-memory DB per test).

## Comment 4 — Default visibility
**My position:** Keeping public default to True on WatchlistEntry. Not switching it to False.

**Reasoning:**  Collection entries have no privacy option at all; they're always public, and they arguably reveal more (you actually watched and rated something) than a watchlist entry does (you just intend to watch something someday). If the more sensitive data is unconditionally public, defaulting the less sensitive data to private would be a strange, backwards story to tell. CineLog bills itself as a community app, and collection's public-by-default, no-toggle design reads like the established posture, not an oversight. Watchlist defaulting to public is consistent with that, and it's actually a step up, since it at least gives users a flag to flip.

**Tradeoff acknowledged:** The real cost isn't the default itself, it's that people won't know about it. Someone who adds a film without reading the docs won't realize it's visible unless they explicitly pass public: false, and watchlist items can be more personal than a rated collection entry. I'm addressing it with the visibility toggle (see stretch features below), so callers can set it explicitly at add time.

## Comment 5 — Sort order
**My position:** I agree the original alphabetical sort was wrong, and I switched get_watchlist to sort by date_added. I went oldest-first instead, and I'd push back on defaulting to newest-first here.

**Reasoning:** Collection sorts newest-first because it's a record of things that already happened, where the most recent entry is the most relevant, same as any activity feed. A watchlist isn't a log, it's a queue: things a user means to get to eventually. Sort it newest-first and every new addition buries whatever's been sitting there for weeks, which is exactly backwards for a list whose whole purpose is not forgetting things. Oldest-first treats it as FIFO, whatever's been waiting longest shows up first, which is what you actually want from a watch-next list.

**Engagement with reviewer's point:** I think the comment bundled two separate claims: alphabetical sorting is arbitrary, and it should specifically match collection for consistency. I agree with the first part completely. I don't think the second follows from it, though , consistency between the two sort orders only matters if collection and watchlist are the same kind of list, and I don't think they are. If there's an actual house rule that every list endpoint returns newest-first, I couldn't find it written down anywhere, and I'd want that decided explicitly before conforming to it. Absent that, oldest-first fits what a watchlist is for.

## Comment 6 — Rebase
**What conflicted:** I ran git fetch origin, then git rebase origin/main. Main had merged a refactor migrating film IDs from integer to UUID (touching Film.id and CollectionEntry.film_id in models.py), plus its own .gitignore addition. Two things came up:

1. An actual conflict on .gitignore — both branches added one, mostly overlapping. Easy fix, just merged the two lists.
2. A silent one in models.py that git didn't flag at all. My old commit that originally added the WatchlistEntry class applied "successfully" against the new main, but enough of the surrounding file had changed that the patch just quietly dropped the WatchlistEntry class instead of erroring. Git reported a clean rebase, but the model was gone, and its film_id — if it had survived — would still have been an integer against the now-UUID Film.id.

**How I resolved it:** Added the WatchlistEntry class back to models.py by hand, with id and film_id as UUID strings to match the rest of the refactor. I also went through the watchlist service and route files for any leftover comments describing film_id as an integer and updated them. Committed this as its own fix commit rather than burying it in the rebase, so the history shows it was a deliberate post-rebase correction.

**How I verified no conflict remains:** I ran the test suite immediately after, which failed with an import error for WatchlistEntry and is what actually surfaced the problem. After restoring the model, I reran everything and got all 13 tests passing. I also checked the branch is linear against main with no merge commits, and grepped for any leftover integer references to the old film_id type.

## Stretch Features

### remove_from_watchlist
Added remove_from_watchlist(user_id, film_id), following the same shape as remove_from_collection: look up the entry, raise a NotInWatchlistError if it's not there, otherwise delete it. Wired up a DELETE endpoint mirroring the collection route. Added tests for both the happy path and the not-present case.

### Second test
I added a test where two different users add the same film to their own watchlists. The dedup check filters by both user_id and film_id together, and it'd be an easy mistake to scope that query by film_id alone, which would silently stop a second, unrelated user from ever adding a popular film. Since WatchlistEntry has no unique constraint at the model level, the application check is the only thing preventing that bug, so it seemed worth locking down with a test rather than assuming it's right.

### Visibility toggle
Added an optional public parameter to add_to_watchlist. Exposed it on the route as an optional field in the add request body, and added tests for both the explicit-false and omitted cases.

## PR Description

### What this feature does
Adds a watchlist to CineLog, a per-user list of films someone means to watch later, separate from their collection of films already watched. Users can add a film, remove one, and view their full watchlist, which comes back in the order films were queued, oldest first. Adding the same film twice is rejected instead of creating a duplicate, and adding a film that doesn't exist returns a 404 instead of a database error.

### Design decisions
- **Default visibility (public=True):** New entries default to public, matching the fact that collection has no privacy option at all. Watchlist actually improves on that by giving users a per-entry flag to turn off. Full reasoning in Comment 4 above.
- **Sort order (oldest first):** get_watchlist returns entries oldest-first, treating the watchlist as a queue rather than an activity log. Full reasoning in Comment 5 above.

