# Mixtape Codebase Map

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
