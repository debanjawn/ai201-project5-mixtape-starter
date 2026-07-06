# Project 5 Submission: Mixtape Bug Hunt

## AI Usage

I used Codex as an in-editor AI helper during codebase orientation and debugging. During Milestone 1, I used it to summarize the project structure, explain the association tables in `models.py`, and trace feature flows from route files into service files and models. I also used ChatGPT to help me understand what the project brief meant by a “codebase map” and to turn my notes into a clearer outline.

I did not use AI as a replacement for reading the code. I verified the structure by opening the files myself, checking the route/service/model connections, running the test suite, and comparing AI explanations against the actual code. As I work through the bug fixes, I will use AI mainly to explain suspicious functions or compare similar code paths, but I will reproduce each bug locally and verify the fix myself before committing.

## Codebase Map

### Overall App Structure

This project is a Flask app organized around routes, services, and SQLAlchemy models.

`app.py` is the Flask application factory. It configures the app, initializes SQLAlchemy, registers the route blueprints, and creates the database tables. The main blueprints are organized by feature area:

- `/songs` from `routes/songs.py`
- `/playlists` from `routes/playlists.py`
- `/users` from `routes/users.py`
- `/feed` from `routes/feed.py`

`models.py` defines the database schema for the app. It contains the main SQLAlchemy models like `User`, `Song`, `Playlist`, `Rating`, `ListeningEvent`, `Notification`, and `Tag`. It also defines association tables for many-to-many relationships.

The `routes/` directory is the HTTP layer. Route files receive requests, read URL parameters or JSON request bodies, call service functions, and return JSON responses with status codes.

The `services/` directory contains the business logic. The README says the bugs live in this layer, so the main debugging strategy is to trace an endpoint from route → service → model instead of only looking at the route.

The `tests/` directory shows the intended behavior of the app. The tests are especially useful because they map closely to specific services:
- `test_streaks.py` checks listening streak behavior
- `test_search.py` checks search behavior
- `test_playlists.py` checks playlist retrieval behavior

`seed_data.py` creates sample users, songs, tags, playlists, friendships, listening events, and notifications. It is useful for understanding how the tables are supposed to connect.

## Models and Important Tables

### User

`User` represents an app user. Important fields include `username`, `email`, `listening_streak`, `last_listened_at`, and `created_at`.

A user is connected to:
- shared songs
- ratings
- listening events
- notifications
- playlists
- friends

This model is important for the listening streak feature and the friends feed feature.

### Song

`Song` represents a song shared in the app. Important fields include `title`, `artist`, `album`, `genre`, `shared_by`, `shared_at`, and `share_note`.

A song is connected to:
- ratings
- listening events
- tags

This model is important for search, ratings, notifications, and playlists.

### Playlist

`Playlist` represents a playlist created by a user. Important fields include `name`, `created_by`, `created_at`, and `is_collaborative`.

A playlist is connected to songs through the `playlist_entries` association table. This matters because playlist song order is not just based on insertion order; the app stores an explicit position for each song.

### ListeningEvent

`ListeningEvent` represents a user listening to a song at a specific time. It connects `user_id`, `song_id`, and `listened_at`.

This model is important for:
- listening streaks
- friends listening now
- activity feed

### Rating

`Rating` represents a user’s 1–5 rating for a song. It stores `user_id`, `song_id`, `score`, and `rated_at`.

There is a unique constraint on `(user_id, song_id)`, meaning one user can only have one rating per song.

### Notification

`Notification` represents a message sent to a user. It stores the recipient user, notification type, body, created time, and read status.

This model is important for friend interaction notifications, such as playlist adds and song ratings.

### Tag

`Tag` represents a label attached to songs. Songs and tags are connected through the `song_tags` association table.

This model matters for search because searches can involve both song fields and tag relationships.

## Association Tables

### friendships

`friendships` connects `User` to `User` as a self-referential many-to-many relationship. It supports the app’s social graph.

The feed service uses this relationship to find a user’s friends, then uses those friend IDs to look up listening activity.

Data flow:
`User` → `friendships` → friend users → `ListeningEvent` → `Song`

### song_tags

`song_tags` connects `Song` to `Tag`. It allows a song to have multiple tags and a tag to belong to multiple songs.

This table is important for search because joining songs to tags can affect the shape of search results.

### playlist_entries

`playlist_entries` connects `Playlist` to `Song`, but it also stores extra information:
- `position`
- `added_by`
- `added_at`

This table is more than a basic join table because `position` controls playlist order. This is important for playlist retrieval because the app should return songs in playlist position order.

## Route Files

### routes/songs.py

This route file handles song-related endpoints, including:
- searching songs
- getting a single song
- rating a song
- recording a listen event

Important service calls include:
- `search_songs`
- `get_song`
- `rate_song`
- `record_listening_event`

Models involved include:
- `Song`
- `Tag`
- `song_tags`
- `Rating`
- `User`
- `ListeningEvent`

### routes/playlists.py

This route file handles playlist-related endpoints, including:
- creating a playlist
- getting playlist details
- getting songs in a playlist
- adding a song to a playlist

Important service calls include:
- `create_playlist`
- `get_playlist`
- `get_playlist_songs`
- `add_to_playlist`

Models and tables involved include:
- `Playlist`
- `Song`
- `User`
- `playlist_entries`
- `Notification` indirectly through playlist add behavior

### routes/users.py

This route file handles user-related endpoints, including:
- getting a user
- getting a user’s streak
- getting notifications
- marking notifications as read

Important service calls include:
- `get_streak`
- `get_notifications`
- `mark_as_read`

Models involved include:
- `User`
- `Notification`

### routes/feed.py

This route file handles feed-related endpoints, including:
- friends listening now
- activity feed

Important service calls include:
- `get_friends_listening_now`
- `get_activity_feed`

Models and tables involved include:
- `User`
- `friendships`
- `ListeningEvent`
- `Song`

## Service Files

### services/streak_service.py

This service owns the listening streak feature.

It reads and writes:
- `User`
- `ListeningEvent`
- `User.listening_streak`
- `User.last_listened_at`

Related routes:
- `POST /songs/<song_id>/listen`
- `GET /users/<user_id>/streak`

This service connects to Issue 1: “My listening streak keeps resetting.”

### services/feed_service.py

This service owns the friends listening now and activity feed features.

It reads:
- `User`
- `User.friends`
- `friendships`
- `ListeningEvent`
- `Song`

Related routes:
- `GET /feed/<user_id>/listening-now`
- `GET /feed/<user_id>/activity`

This service connects to Issue 2: “Friends Listening Now shows people from yesterday.”

### services/search_service.py

This service owns song search and song detail retrieval.

It reads:
- `Song`
- `song_tags`
- `Tag`

Related routes:
- `GET /songs/search`
- `GET /songs/<song_id>`

This service connects to Issue 3: “The same song keeps showing up twice in search.”

### services/notification_service.py

This service owns notification behavior and some song/playlist interaction side effects.

It reads and writes:
- `Notification`
- `Song`
- `User`
- `Rating`
- `Playlist`
- `playlist_entries` indirectly through playlist song relationships

Related routes:
- `POST /songs/<song_id>/rate`
- `GET /users/<user_id>/notifications`
- `POST /users/notifications/<notification_id>/read`
- `POST /playlists/<playlist_id>/songs`

This service connects to Issue 4: “I got notified when a friend added my song to a playlist but not when they rated it.”

### services/playlist_service.py

This service owns playlist creation and playlist retrieval.

It reads and writes:
- `Playlist`
- `User`
- `Song`
- `playlist_entries`

Related routes:
- `POST /playlists/`
- `GET /playlists/<playlist_id>`
- `GET /playlists/<playlist_id>/songs`

This service connects to Issue 5: “The last song in a playlist never shows up.”

## Real Data Flow: Viewing Playlist Songs

Feature: viewing all songs in a playlist.

Endpoint:
`GET /playlists/<playlist_id>/songs`

Flow:
1. The request enters `routes/playlists.py`.
2. The route function receives `playlist_id` from the URL.
3. The route calls `get_playlist_songs(playlist_id)` in `services/playlist_service.py`.
4. The service checks whether the playlist exists.
5. The service queries `Song` through the `playlist_entries` association table.
6. The query filters by `playlist_entries.playlist_id`.
7. The query orders songs using `playlist_entries.position`.
8. Each song is serialized with `Song.to_dict()`.
9. The route returns JSON with:
   - `songs`
   - `count`

Models and tables involved:
- `Playlist`: used to confirm the playlist exists
- `Song`: the main object returned
- `playlist_entries`: connects playlists to songs and stores the order

Expected behavior:
The endpoint should return every song in the playlist in the correct playlist order.

## Real Data Flow: Rating a Song

Feature: user rates a song.

Endpoint:
`POST /songs/<song_id>/rate`

Flow:
1. The request enters `routes/songs.py`.
2. The route reads `song_id` from the URL.
3. The route reads `user_id` and `score` from the request body.
4. The route calls `rate_song(user_id, song_id, score)` in `services/notification_service.py`.
5. The service validates the score.
6. The service loads the `Song`.
7. The service loads the `User`.
8. The service checks whether a `Rating` already exists for this user/song pair.
9. The service creates or updates the `Rating`.
10. The route returns the rating as JSON.

Models involved:
- `Song`
- `User`
- `Rating`
- potentially `Notification`

Expected behavior:
The user’s rating should be saved, and any required notification side effects should happen.

## Real Data Flow: Friends Listening Now

Feature: show what friends are currently listening to.

Endpoint:
`GET /feed/<user_id>/listening-now`

Flow:
1. The request enters `routes/feed.py`.
2. The route receives `user_id` from the URL.
3. The route calls `get_friends_listening_now(user_id)` in `services/feed_service.py`.
4. The service loads the `User`.
5. The service gets friend IDs through the `User.friends` relationship and `friendships` table.
6. The service queries `ListeningEvent` for those friends.
7. The service loads related `Song` and friend `User` data.
8. The route returns a list of listening activity.

Models and tables involved:
- `User`
- `friendships`
- `ListeningEvent`
- `Song`

Expected behavior:
The feed should only show relevant recent listening activity from the user’s friends.

## Patterns I Noticed

The app follows a route → service → model pattern.

Routes are mostly thin. They handle request data, call service functions, and format JSON responses.

Services are thicker. They contain validation, database queries, updates, and most of the business rules.

Models define the database structure and provide `to_dict()` helpers for JSON serialization.

Association tables are important to how the app works:
- `friendships` powers the social graph
- `song_tags` powers tag metadata and search behavior
- `playlist_entries` powers playlist membership and ordering

The app mixes two styles of many-to-many access:
- ORM relationship access, such as using relationship fields
- direct association table queries, especially when extra fields like `position` matter

The tests are useful for understanding intended behavior because each test file maps closely to a service area.

## Planned Bug Focus

After reading the README, models, routes, services, and tests, I chose to focus on these three issue areas first.

### Issue 1: My listening streak keeps resetting

Service:
`services/streak_service.py`

Related models:
- `User`
- `ListeningEvent`

Related fields:
- `User.listening_streak`
- `User.last_listened_at`
- `ListeningEvent.listened_at`

Related routes:
- `POST /songs/<song_id>/listen`
- `GET /users/<user_id>/streak`

Why I chose this:
The test suite already includes a failing case around streak behavior, so this is a good issue to investigate through the existing tests and service logic.

### Issue 3: The same song keeps showing up twice in search

Service:
`services/search_service.py`

Related models/tables:
- `Song`
- `Tag`
- `song_tags`

Related route:
- `GET /songs/search`

Why I chose this:
Search involves joins between songs and tags, and duplicate rows can happen when a query joins across many-to-many relationships. This makes it a good issue for practicing tracing query behavior.

### Issue 5: The last song in a playlist never shows up

Service:
`services/playlist_service.py`

Related models/tables:
- `Playlist`
- `Song`
- `playlist_entries`

Related route:
- `GET /playlists/<playlist_id>/songs`

Why I chose this:
Playlist retrieval has a clear route → service → association table flow. The `playlist_entries.position` field controls ordering, so this issue is useful for understanding how ordered many-to-many data is retrieved.

---

# Root Cause Analysis Entries

## Issue 1: My listening streak keeps resetting

### How I reproduced it

I reproduced this by running `python -m pytest tests/test_streaks.py -v` before changing any code. The Sunday-specific test failed. The test listened on Saturday and then Sunday. After Saturday, the streak was 1. After Sunday, the expected streak was 2, but the actual streak stayed at 1. This confirmed that the streak failed to increment across the Saturday-to-Sunday boundary.

### How I found the root cause

I traced the failing test to `update_listening_streak()` in `services/streak_service.py`. The function calculates `days_since_last` by comparing today’s date to `user.last_listened_at`. The main consecutive-day condition was on line 73: `days_since_last == 1 and today.weekday() != 6`. Since the failing test specifically involved Sunday, I checked Python’s `weekday()` behavior and saw that Sunday is `6`.

### The root cause

The streak logic correctly checked whether the user listened yesterday with `days_since_last == 1`, but it also excluded Sundays with `today.weekday() != 6`. On Sunday, `today.weekday()` returns `6`, so the consecutive-day condition became false even when the user had listened on Saturday. The code then fell into the reset branch and set the streak back to 1 instead of incrementing it.

### My fix and side-effect check

I removed the unnecessary Sunday exclusion so the streak increments whenever `days_since_last == 1`, including Saturday-to-Sunday. I verified the fix by running `python -m pytest tests/test_streaks.py -v`. All five streak tests passed, including starting a streak, incrementing on consecutive days, avoiding double-counting on the same day, resetting after a skipped day, and incrementing on Sunday.

## Issue 3: The same song keeps showing up twice in search

### How I reproduced it

I investigated this by running manual search requests against seeded multi-tag songs, including `Crown Heights`, `Harlem`, `After Hours`, `Lagos`, and `Frequencies`. These each returned `count: 1`, so they did not reproduce the duplicate-result bug yet. I also tried tag-only searches like `rap` and `hip-hop`, which returned empty results. Next I need to inspect `services/search_service.py` and `tests/test_search.py` to find the exact input or data condition that triggers duplicates.

### How I found the root cause

TODO: Describe which files/functions I traced and what led me to the specific root cause.

### The root cause

TODO: Explain the precise bug in plain English.

### My fix and side-effect check

TODO: Explain what I changed, why it fixed the issue, and what related behavior I checked afterward.

## Issue 5: The last song in a playlist never shows up

### How I reproduced it

I reproduced this by running `python -m pytest tests/test_playlists.py -v` before changing any code. Two playlist tests failed. `test_playlist_returns_all_songs` expected `get_playlist_songs()` to return 5 songs, but it returned 4. `test_playlist_returns_songs_in_order` expected Track 1 through Track 5, but the actual result only included Track 1 through Track 4. This confirmed that the final playlist song was missing.

### How I found the root cause

I traced the failing tests to `get_playlist_songs()` in `services/playlist_service.py`. The query loaded songs through `playlist_entries`, filtered by playlist ID, and ordered by `playlist_entries.position`, which matched the expected data flow. The bug was not in the query. The suspicious line was the final return statement, which converted `songs[:-1]` into dictionaries. That slice removes the last item from the list.

### The root cause

The service correctly queried all songs in the playlist, but the return statement used `songs[:-1]`. In Python, `[:-1]` returns the list without its final element. As a result, every playlist response intentionally dropped the last song, even though the database query had loaded it correctly.

### My fix and side-effect check

I changed the return statement to iterate over `songs` instead of `songs[:-1]`, so every queried song is serialized and returned. I verified the fix by running `python -m pytest tests/test_playlists.py -v`. All playlist tests passed, including the test for returning all 5 songs, the test for preserving song order, and the empty playlist test.