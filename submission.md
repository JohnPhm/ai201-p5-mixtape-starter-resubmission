# Project 5: Mixtape Bug Hunt — Submission

## Phase 1: Codebase Map

Mixtape is a Flask + SQLAlchemy JSON API for a social music app: users share songs, listen to them, rate them, build collaborative playlists, and get notified when friends interact with songs they shared. No frontend, no auth — every endpoint takes the acting user's ID in the URL or the JSON body.

---

### Main files

There are two main files for this project. The first one is app.py, which is the app factory. "create_app(config=None)" configures SQLAlchemy (SQLite by default, overridable so tests can pass an in-memory DB), registers four blueprints under "/songs",
"/playlists", "/users", "/feed", and runs "db.create_all()".

The second one is models.py, which is comprised of 7 models (User, Tag, Song, ListeningEvent, Rating, Playlist, Notification) with string UUID primary keys and UTC timestamps. Each one defines to_dict(), which is the app's JSON shape. 

Three association tables:   
- `friendships` — self-referential, and **not** automatically symmetric: a mutual friendship needs two rows (`seed_data.py` inserts both). `User friends` is `lazy="dynamic"`, so it's a query, not a list.
- `song_tags` — plain many-to-many between songs and tags.
- `playlist_entries` — a join table with extra columns: `position` and `added_by` (both `nullable=False`) plus `added_at`. A song's place in a playlist is an explicit integer, not insertion order. But `Playlist.songs` is a plain relationship that knows nothing about those columns, so anything needing order must query `playlist_entries` directly.

Two data-model details that drive features: `Song.shared_by` is what the entire notification system keys off ("notify whoever shared this song"), and `User.listening_streak` / `last_listened_at` are denormalized onto the user row — the streak is written at listen time, never recomputed from `ListeningEvent` history.

**`routes/`** — Four blueprints, all thin: parse input, call one service, jsonify.

| File | Endpoints |
|---|---|
| `songs.py` | `GET /songs/search?q=`, `GET /songs/<id>`, `POST /songs/<id>/rate`, `POST /songs/<id>/listen` |
| `playlists.py` | `POST /playlists/`, `GET /playlists/<id>`, `GET/POST /playlists/<id>/songs` |
| `users.py` | `GET /users/<id>`, `/streak`, `/notifications`, `POST /users/notifications/<id>/read` |
| `feed.py` | `GET /feed/<id>/listening-now`, `GET /feed/<id>/activity` |

**`services/`** — All business logic, one module per feature area.

- `streak_service.py` — `record_listening_event()` creates the event, then
  `update_listening_streak()` branches on the calendar-day gap (0 = no change, 1 = increment,
  more = reset to 1). It re-attaches `timezone.utc` to naive datetimes read back from SQLite.
- `feed_service.py` — `get_friends_listening_now()` queries friends' events newer than
  `RECENT_THRESHOLD` (a module constant, currently 24 hours), sorts newest-first, and dedupes to
  one entry per friend via a `seen_friends` set. `get_activity_feed()` is the same query with no
  recency filter and no dedupe, capped at 20.
- `search_service.py` — `search_songs()` outer-joins `song_tags` and applies an `ilike` OR-filter
  across title and artist. Tag names actually come from `Song.tags` (`lazy="subquery"`), not the
  join. `get_song()` is a PK lookup that raises `ValueError` when missing.
- `notification_service.py` — Owns `create_notification()` *and* the two writes meant to trigger
  notifications: `add_to_playlist()` (appends the song, then notifies `song.shared_by` unless
  they're the one who added it) and `rate_song()` (validates 1–5, then updates an existing
  `Rating` or inserts a new one — `Rating` has a unique constraint on user+song). Read side:
  `get_notifications()`, `mark_as_read()`.
- `playlist_service.py` — `create_playlist()`, `get_playlist()` (metadata only),
  `get_user_playlists()`, and `get_playlist_songs()`, which joins `playlist_entries`, orders by
  `position`, and returns `[song.to_dict() for song in songs[:-1]]`.

**`seed_data.py`** — Drops and rebuilds everything: 5 users, 5 bidirectional friendships,
13 songs deliberately split into 0-tag / 1-tag / 3+-tag groups, listening events at mixed ages
(~10 min old vs. 2 hours to 2.5 days old), `last_listened_at` for three users, 3 playlists
inserted directly into `playlist_entries` with explicit `position`, and one example notification.
The data is shaped to make specific behaviors observable.

**`tests/`** — `test_streaks.py`, `test_search.py`, `test_playlists.py`. Each builds an
in-memory app fixture and calls service functions directly, so the tests exercise the service
layer rather than HTTP.

### Data flow: adding a song to a playlist

`POST /playlists/<id>/songs` with `{"song_id", "added_by"}`

1. `routes/playlists.py::add_song()` checks both fields are present (400 if not) and calls
   `notification_service.add_to_playlist()` — note the playlist *write* lives in the
   notification service, not `playlist_service`.
2. That function loads the song, adder, and playlist, raising `ValueError` on any miss.
3. If the song isn't already present, it appends to `playlist.songs` and commits. Because that
   goes through the plain relationship, the `playlist_entries` row gets only the two FKs — no
   `position`, no `added_by`.
4. If `song.shared_by != added_by_user_id`, it calls `create_notification()` for the sharer with
   type `song_added_to_playlist`.
5. The route returns `{"message": ...}`, 201 — no playlist state, so the caller must re-fetch.

The recipient reads it later via `GET /users/<id>/notifications` → `get_notifications()`.

### Patterns I noticed

- **Routes are thin adapters; services own everything.** One exception:
  `routes/users.py::get_user()` queries `User` directly instead of calling a service.
- **`ValueError` is the only error channel.** Services raise it with a readable message; routes catch it and map it by route *intent*, not exception type — lookups return 404, mutations 400. There are no custom exception classes.
- **Services group by user action, not by model.** Rating lives in `notification_service` because rating is supposed to generate a notification. "Where does this live?" is answered by what the user did, not what table changed.
- **`to_dict()` is the serialization boundary** — defined once per model, so services return plain dicts. Only `rate_song()` returns an ORM object and lets the route serialize it.
- **Time is UTC-aware on write but comes back naive from SQLite,** so any timestamp comparison has to re-attach the timezone first. `update_listening_streak()` does this explicitly.
- **Streak state is denormalized** — the event log and the counter can drift, since nothing recomputes one from the other.

### Reproducing Errors
1. Listening streak keeps resetting
To trigger the bug, we can use the pytest file test_streaks.py. The command "pytest tests/test_streaks.py::test_streak_increments_on_sunday" will use the streak incrementing on Sunday. This test results in an AssertionError, suggesting that it failed and that it is not incrementing correctly. 

Looking at the error, the AssertionError pointed at tests/test_streaks.py:96, so I opened that test and saw it imports update_listening_streak from services/streak_service.py and calls it directly. This tells me that the failure is found within the service, not the route. I opened streak_service.py and read update_listening_streak() top to bottom against its own docstring, which lists four rules: no prior history, same day, consecutive day, gap. The docstring's third rule says "if the user listened yesterday: streak increments by 1", but the code's matching branch had a second condition that the docstring never mentions. 

Bug Fix for Issue #1:
The faulty logic was in update_listening_streak() in services/streak_service.py. The branch that increments the streak on a consecutive day had an extra condition tacked on:
    elif days_since_last == 1 and today.weekday() != 6:
        user.listening_streak += 1

In Python, datetime.weekday() returns 6 for Sunday. So whenever "today" was a Sunday, the increment branch was skipped even though the user had listened the day before (Saturday). Control then fell through to the else branch, which resets the streak to 1. That is exactly why Kenji's streak was wiped every Sunday morning and then started counting again from Monday. This extra weekday check has no basis in the streak rules — a consecutive day is a consecutive day regardless of which day of the week it is.

The fix was to remove the "and today.weekday() != 6" clause so the branch simply increments whenever exactly one day has passed.

Side-effect check: all 5 tests in test_streaks.py pass, including the four that were already passing: new user starts at 1, consecutive days increment, same-day listens don't double count, and a skipped day still resets to 1. TThe last one matters the most, since it proves removing the condition didn't break the reset path; the clause I deleted only guarded the increment branch, so gap handling was never affected. I also confirmed get_streak() and GET /users/<id>/streak were untouched.



2. Friends Listening Now shows people from yesterday
"Friends Listening Now" is supposed to show me what my friends are playing right now — or at least what they've played today. This morning around 9am it showed darius "listening now" to a song he told me he played at 11pm last night, before he went to bed. He hadn't opened the app all morning. Stuff from yesterday evening keeps hanging around in the feed until the same time the next day.

Steps I took to find the root cause of the bug:
I started at the route, since the issue report names the endpoint. GET /feed/<user_id>/listening-now is handled in routes/feed.py, which is simply a file that calls get_friends_listening_now(user_id), catches ValueErrors, and turns the results into JSON. I then decided to look at services/feed_service.py instead. get_friends_listening_now() does four things: load the user, compute a cutoff, query events filtered by listened_at >= cutoff, and dedupe to one entry per friend. I ruled out the last three. The dedupe keeps each friend's newest event because the query is ordered desc(listened_at), so it can't resurrect an older listen. This meant that the cutoff was the only thing left deciding whether a stale event shows up. 

Expected: only friends who have listened today appear. Actual: friends whose last listen was yesterday evening still show up the next morning.

Bug fix for issue #2:
The faulty line was in the cutoff mentioned in get_friends_listening_now(). RECENT_THRESHOLD was timedelta(hours=24), and the cutoff was computed as datetime.now(timezone.utc) - RECENT_THRESHOLD. That's a rolling 24-hour window, not a calendar day. The feed was answering "who listened in the last 24 hours" when the feature is called "Listening Now." Nothing about the query, the ordering, or the per-friend dedup was wrong — a single constant encoded the wrong notion of recency.

Side-effect check: 
I replaced the rolling cutoff with the start of the current UTC day, so the filter now means "listened today" instead of "listened in the last 24 hours." get_activity_feed() remains unaffected since it never used the cutoff. The rest of the function is untouched as the query still orders newest-first and the dedupe still keeps one entry per friend, so the feed's shape and ordernig are unchanged. 

3. The same song keeps showing up twice in search
When I search, some songs come back two or even three times — identical entries, same song. I searched "Anthem" and Crown Heights Anthem by Borough Kings showed up three times in the results. Other songs only show up once. Nothing about the duplicates looks different; it's just the same result repeated.

How I reproduced the error:
I seeded the database and ran GET /songs/search?q=Anthem, the same request simone described. It returned count: 1, meaning the reported symptom did not reproduce. Since Crown Heights Anthem has 3 tags in the seed data and was the exact song in the report, I checked whether the join was multiplying rows anyway by running the same query three ways: the raw SQL statement returned 3 rows, query(Song).all() returned 1, and the 2.0-style select(Song)...scalars().all() returned 3. That confirmed the duplication is real in the query but is being discarded before it reaches the response. All 5 tests in test_search.py also passed before I changed anything, which matches that result.

How I found root cause of bug:
The codebase map pointed me at services/search_service.py. search_songs() is a single query, so I read it line by line and found .outerjoin(song_tags, Song.id == song_tags.c.song_id). I then checked whether anything downstream actually used it — nothing filters, orders, or selects on tags, and the tags list in the response comes from the Song.tags relationship in to_dict(), not from this join. A join whose only effect is on the number of rows returned was the specific cause, not just a suspicious line.

Bug fix for issue #3:
The query did an outerjoin with song_tags against the song-tags join table. Because a song can have multiple tags, the join produces one row per (song, tag) pair. This means that a song with 3 tags came back as 3 identifcal song rows while a song with one tag appeared only once. This was fixed by removing the unnecessary outerjoin so that the query returns one row per matching song. 