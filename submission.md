# Mixtape Codebase Map

## AI Usage
I used AI tools as a debugging assistant while working through the bug fixes and the submission writeup. I asked it to trace the flow from routes into services, summarize what each service was supposed to return, and explain why certain bugs were happening, such as duplicate search results, the missing rating notification, and the playlist entry that was getting dropped. It also helped me turn those observations into the Bug Reproduction Notes and Root Cause Analysis sections in a consistent format.

I still verified the important details myself by reading the actual model definitions, service code, and tests, then checking the behavior against the test suite. In a few cases the AI was helpful but not sufficient on its own: for example, I had to confirm the notification behavior in `notification_service.py` against the route and model flow, and I had to check the playlist and search bugs against their tests to make sure the explanation matched the real failure. When the AI suggested an interpretation that was too broad or left an edge case unclear, I treated it as a lead and confirmed the final answer by reading the code directly.

## Project Overview
Mixtape is a Flask social music app. Users can share songs, build collaborative playlists, track listening streaks, view friends' recent listening activity, and receive notifications when other people interact with their shared songs.

The app is organized around a simple pattern: routes handle HTTP requests, services contain the business logic and database queries, and models define the data structure. The entry point wires everything together through a Flask app factory.

## Main Files

### `app.py`
This is the application entry point. It creates the Flask app, configures the database connection, initializes SQLAlchemy, registers all blueprints, and creates the database tables when the app starts.

### `models.py`
This file defines the database schema. It contains the core entities: `User`, `Song`, `Tag`, `Playlist`, `ListeningEvent`, `Rating`, and `Notification`. It also defines the association tables used for many-to-many relationships:

- `friendships` for user-to-user friend links
- `song_tags` for song/tag relationships
- `playlist_entries` for ordered songs inside playlists

Each model also includes a `to_dict()` method so routes and services can serialize records to JSON consistently.

### `routes/songs.py`
This blueprint exposes song-related endpoints:

- `GET /songs/search` searches songs by title or artist
- `GET /songs/<song_id>` returns a single song
- `POST /songs/<song_id>/rate` saves a rating
- `POST /songs/<song_id>/listen` records a listening event and updates streaks

### `routes/playlists.py`
This blueprint handles playlist requests:

- `POST /playlists/` creates a playlist
- `GET /playlists/<playlist_id>` returns playlist metadata
- `GET /playlists/<playlist_id>/songs` returns songs in playlist order
- `POST /playlists/<playlist_id>/songs` adds a song to a playlist

### `routes/users.py`
This blueprint serves user-facing data:

- `GET /users/<user_id>` returns a user profile
- `GET /users/<user_id>/streak` returns the listening streak
- `GET /users/<user_id>/notifications` lists notifications
- `POST /users/notifications/<notification_id>/read` marks a notification as read

### `routes/feed.py`
This blueprint powers the social feed views:

- `GET /feed/<user_id>/listening-now` shows friends who listened recently
- `GET /feed/<user_id>/activity` shows a longer recent activity feed

### `services/search_service.py`
Implements song lookup logic. It searches by title or artist and returns serialized song data.

### `services/playlist_service.py`
Implements playlist creation and retrieval. It validates the creator, creates playlists, fetches a playlist's metadata, and returns songs in playlist order.

### `services/notification_service.py`
Implements notification behavior. It creates notifications, returns notifications for a user, marks notifications as read, and creates a notification when someone adds a shared song to a playlist.

### `services/streak_service.py`
Handles listening streak logic. It records listening events, updates the user's streak based on the date of the last listen, and returns the current streak.

### `services/feed_service.py`
Builds the social feed views. It looks at a user's friends, finds recent listening events, deduplicates the "listening now" feed to the most recent event per friend, and returns activity lists ordered by recency.

### `seed_data.py`
Populates the database with sample users, songs, tags, friendships, playlists, listening events, and notifications. It is useful for seeing the app in a realistic state and for understanding how the relationships are supposed to look.

### `tests/`
The tests focus on the bug-prone service logic:

- `test_streaks.py` checks listening streak behavior
- `test_search.py` checks that song search does not duplicate results
- `test_playlists.py` checks playlist ordering and completeness

## Example Data Flow: Adding a Shared Song to a Playlist
One concrete flow in the app is adding a shared song to a playlist, which can trigger a notification for the original sharer.

1. A client sends `POST /playlists/<playlist_id>/songs` with `song_id` and `added_by`.
2. `routes/playlists.py` validates the request body and calls `services.notification_service.add_to_playlist()`.
3. `add_to_playlist()` loads the `Song`, `User`, and `Playlist` records from the database.
4. If the song is not already in the playlist, it appends the song and commits the change.
5. If the person adding the song is not the original sharer, the service creates a `Notification` for `song.shared_by`.
6. `routes/users.py` can later expose that notification through `GET /users/<user_id>/notifications`.

The same style applies to the listening flow: `POST /songs/<song_id>/listen` calls `record_listening_event()`, which creates a `ListeningEvent` and updates the user's listening streak.

## Organization Patterns
The app follows a few consistent patterns throughout the repo:

- Blueprints are grouped by feature area (`songs`, `playlists`, `users`, `feed`).
- Routes stay thin and mostly translate request data into service calls.
- Services own validation, database lookups, and transaction commits.
- Models centralize schema, relationships, and JSON serialization.
- Errors are usually raised as `ValueError` in services and converted to HTTP responses in routes.
- UUID string primary keys are used across the database instead of incremental integers.
- Many-to-many relationships are modeled explicitly with association tables so the app can store extra metadata such as playlist position and who added a song.

## Overall Structure
The project is intentionally split so that each feature can be traced from endpoint to service to model. That makes it easier to reason about bugs, test individual behaviors, and keep the API layer separate from the database logic.

## Bug Reproduction Notes

### Issue #1: My listening streak keeps resetting
**How I reproduced it:** I set a user’s `last_listened_at` to Saturday and then recorded another listen on Sunday, using the same user and two consecutive calendar dates. The expected behavior was that the streak would increment from 1 to 2, but the app treated Sunday as a reset condition and returned a streak of 1.

**Trigger condition:** This only shows up when the previous listen was exactly one day earlier and the new listen happens on Sunday.

### Issue #3: The same song keeps showing up twice in search
**How I reproduced it:** I used a song that has multiple tags in the seed/test data, then searched for that song by title. The query returned the same song multiple times because the search joins through the tag table, which multiplies rows for songs with more than one tag.

**Trigger condition:** A song with multiple tags must match the search terms by title or artist.

### Issue #4: I got notified when a friend added my song to a playlist but not when they rated it
**How I reproduced it:** I had another user rate one of my shared songs and then checked my notifications. The app created notifications for playlist additions, but the rating action did not create a new notification, so the event never appeared in my notification list.

**Trigger condition:** A different user rates a song that I shared.

### Issue #5: The last song in a playlist never shows up
**How I reproduced it:** I created a playlist with several songs and requested its song list through the playlist endpoint. The response consistently omitted the final song in the playlist, so a 5-song playlist came back with only 4 songs.

**Trigger condition:** Any playlist with at least one song exposes the bug, because the service slices off the last item before returning results.

# Bug Root Cause Analysis

### Issue #1: My listening streak keeps resetting
1. **Issue number and title**
	Issue #1: My listening streak keeps resetting

2. **How you reproduced it**
	I used the streak test setup and a controlled pair of dates. First I updated a user's streak with a Saturday timestamp, then I updated the same user again with a Sunday timestamp. That reproduces the reported bug because the second listen is exactly one calendar day later and crosses the Saturday-to-Sunday boundary that was failing.

3. **How you found the root cause**
	I started from [tests/test_streaks.py](/Users/hiennguyen/CodePath/ai201-project5-mixtape-starter/tests/test_streaks.py) because the failing case was already encoded there as the Sunday test. From there I opened [services/streak_service.py](/Users/hiennguyen/CodePath/ai201-project5-mixtape-starter/services/streak_service.py) and traced `update_listening_streak()`. The moment that made the cause clear was the conditional `days_since_last == 1 and today.weekday() != 6`, because that explicitly treated Sunday as a non-consecutive day even when the dates were one day apart.

4. **The root cause**
	The streak update logic had an extra Sunday exception baked into the consecutive-day check. Instead of incrementing whenever the previous listen was exactly one day ago, it refused to increment when the new day was Sunday, so a valid Saturday-to-Sunday sequence was incorrectly reset.

5. **Your fix and side-effect check**
	I removed the Sunday-only exclusion so any one-day gap now increments the streak. That fixes the root cause because the logic once again matches the stated rule: consecutive calendar days should count, regardless of weekday. After the change, I ran [tests/test_streaks.py](/Users/hiennguyen/CodePath/ai201-project5-mixtape-starter/tests/test_streaks.py), and the full streak suite passed, which also confirmed that same-day no-change and skipped-day reset behavior still worked.

### Issue #3: The same song keeps showing up twice in search
1. **Issue number and title**
   Issue #3: The same song keeps showing up twice in search

2. **How you reproduced it**
   I used a song with multiple tags in the search test data and searched for that song by title. The response returned the same song more than once because each matching tag row created another copy of the song in the query results.

3. **How you found the root cause**
   I started from [tests/test_search.py](/Users/hiennguyen/CodePath/ai201-project5-mixtape-starter/tests/test_search.py) because the duplicate-result behavior was already captured there with a multi-tag song. From there I traced the search flow into [services/search_service.py](/Users/hiennguyen/CodePath/ai201-project5-mixtape-starter/services/search_service.py). The key clue was the query structure: it joined through song_tags, which expands one Song row into multiple rows when that song has more than one tag.

4. **The root cause**
   The search query was joining the song table to the song-tag association table even though the response only needed matched songs and their serialized tags. That join multiplied rows for songs with multiple tags, so one logical song became several SQL result rows and then several JSON entries.

5. **Your fix and side-effect check**
   I removed the unnecessary join so the query returns each matching song only once and still relies on Song.to_dict() to include tags. That fixes the duplication at the source instead of trying to deduplicate the response afterward. After the change, I ran [tests/test_search.py](/Users/hiennguyen/CodePath/ai201-project5-mixtape-starter/tests/test_search.py), and the search suite passed, including the multi-tag case.

### Issue #4: I got notified when a friend added my song to a playlist but not when they rated it
1. **Issue number and title**
	Issue #4: I got notified when a friend added my song to a playlist but not when they rated it

2. **How you reproduced it**
	I had another user rate one of my shared songs and then checked my notifications. The playlist-add notification flow worked, but the rating action did not create a new notification, so nothing showed up for the rating event.

3. **How you found the root cause**
	I started from [README.md](README.md) and the rating route in [routes/songs.py](/Users/hiennguyen/CodePath/ai201-project5-mixtape-starter/routes/songs.py) because the bug was tied to POST /songs/<song_id>/rate. That led me into [services/notification_service.py](/Users/hiennguyen/CodePath/ai201-project5-mixtape-starter/services/notification_service.py), where `rate_song()` handled the rating write. The missing piece was that the function saved the rating but did not create a corresponding notification for the song owner.

4. **The root cause**
	The rating service only persisted the `Rating` record and stopped there. Unlike the playlist-add flow, it had no call to `create_notification()` for the song sharer, so the notification table never received a `song_rated` entry when someone else rated the song.

5. **Your fix and side-effect check**
	I added a `song_rated` notification after the rating commit so the owner is notified when another user rates their shared song. That fixes the root cause because the rating workflow now mirrors the existing playlist notification pattern instead of ending after the database update. After the change, I rechecked the relevant service flow and confirmed the rating path now creates the notification as expected.

### Issue #5: The last song in a playlist never shows up
1. **Issue number and title**
	Issue #5: The last song in a playlist never shows up

2. **How you reproduced it**
	I created a playlist with several songs and requested its songs through the playlist endpoint. The response returned every song except the last one, so a 5-song playlist came back with only 4 songs.

3. **How you found the root cause**
	I started from [tests/test_playlists.py](/Users/hiennguyen/CodePath/ai201-project5-mixtape-starter/tests/test_playlists.py) because the failing behavior was already described there. From there I traced the request into [services/playlist_service.py](/Users/hiennguyen/CodePath/ai201-project5-mixtape-starter/services/playlist_service.py) and checked the return path in `get_playlist_songs()`. The issue was in the final return statement, which had an exception that removed the last item before sending the list back.

4. **The root cause**
	The playlist retrieval logic was collecting all songs in the correct order, but then it applied a last-song exception when returning the result. That meant the service was intentionally dropping the final playlist entry even though the query itself had already loaded the full playlist.

5. **Your fix and side-effect check**
	I removed the last-song exception so `get_playlist_songs()` now returns the complete ordered list. That fixes the bug at the source because the service no longer truncates valid data after querying it. After the change, I ran [tests/test_playlists.py](/Users/hiennguyen/CodePath/ai201-project5-mixtape-starter/tests/test_playlists.py), and the playlist suite passed, including the order and empty-playlist cases.