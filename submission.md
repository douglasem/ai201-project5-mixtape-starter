# Mixtape Bug Hunt Submission

## AI Usage

I used ChatGPT/Claude to help with codebase orientation and debugging. I asked for explanations of unfamiliar service functions, call chains, and edge cases after I had already located relevant files myself. I verified AI suggestions by reading the code, reproducing bugs manually, and testing behavior before committing fixes.

## Codebase Map

### Main Files and Roles

- `app.py`: creates the Flask app and registers routes.
- `models.py`: defines database models such as User, Song, Playlist, PlaylistSong, and Notification.
- `routes/`: contains HTTP endpoint handlers. Routes parse requests, call services, and return JSON.
- `services/`: contains application logic such as search, playlist handling, stats, feeds, and notifications.
- `seed_data.py`: creates test users, songs, playlists, and sample app state.

### Data Flow Example: Sharing a Song

A user action starts in a route file, which receives the request and extracts parameters. The route calls a service function that updates the database and creates related records such as notifications or feed entries. The service commits the change, and the route formats the result as JSON.

## Root Cause Analyses


## Issue #4: Rating a shared song does not notify the original sharer

### How I reproduced it
I traced the rating feature from the route to the service implementation. The `POST /songs/<song_id>/rate` route in `routes/songs.py` calls `rate_song()` in `services/notification_service.py`.

I then compared the rating flow to the existing playlist notification flow. The `add_to_playlist()` function successfully creates a notification for the original song sharer when another user adds their song to a playlist. In contrast, `rate_song()` validates the request, creates or updates the rating, commits it to the database, and returns without creating any notification. This matches the reported behavior that playlist notifications work but rating notifications do not.

### How I found the root cause
I began in `routes/songs.py` to see how song ratings were handled. From there I followed the call into `services/notification_service.py`.

Within the same file I compared two similar features:

```
POST /songs/<song_id>/rate
    ↓
rate_song()

vs.

POST /playlists/<playlist_id>/songs
    ↓
add_to_playlist()
    ↓
create_notification()
```

The comparison made the missing step obvious. The playlist feature called `create_notification()` after updating the database, while the rating feature never did.

### The root cause
The `rate_song()` function successfully saved the rating but never generated a notification for the original song sharer. Unlike `add_to_playlist()`, it omitted the call to `create_notification()`, so users were never notified when someone rated one of their shared songs.

### My fix and side-effect check
I added a notification after the rating is saved. If someone other than the original sharer rates the song, `rate_song()` now calls `create_notification()` using the `song_rated` notification type.

After making the change, I reran the existing test suite with:

```bash
pytest
```

All existing tests continued to pass, confirming that the notification change did not affect playlist behavior, search, or streak functionality.

### AI usage
After tracing the call chain myself, I used ChatGPT to compare the structure of `rate_song()` and `add_to_playlist()`. The comparison highlighted that the playlist flow called `create_notification()` while the rating flow did not. I verified this by reading both functions before implementing the fix.


## Issue #5: The last song in a playlist never shows up

### How I reproduced it
I ran `pytest tests/test_playlists.py`. The playlist tests showed that a playlist seeded with 5 songs only returned 4 songs, and the ordered title list stopped at `"Track 4"` instead of including `"Track 5"`.

### How I found the root cause
I traced the playlist route from `routes/playlists.py` to `services/playlist_service.py`, specifically `get_playlist_songs()`. The database query correctly joined `Song` to `playlist_entries`, filtered by playlist ID, and ordered by `playlist_entries.c.position`. That made me confident the query was returning the right ordered data. The specific problem was the return statement after the query.

### Root cause
`get_playlist_songs()` used `songs[:-1]` when converting songs to dictionaries. In Python, `[:-1]` means “all items except the last one,” so the function always dropped the final playlist song even though the query had retrieved it correctly.

### Fix and side-effect check
I changed the return statement to iterate over `songs` instead of `songs[:-1]`, so every queried song is returned. I verified the fix by rerunning `pytest tests/test_playlists.py`, which checks both that all 5 songs are returned and that they remain in playlist position order. I also reran the full test suite with `pytest` after committing the fix. All tests passed except the unrelated streak test, confirming that returning the complete playlist did not affect search, notifications, or other playlist functionality.

### AI usage
After tracing the playlist route to `get_playlist_songs()`, I used ChatGPT to explain the behavior of Python list slicing. It confirmed that `songs[:-1]` returns every element except the last one. I verified this directly in the code before changing the return statement.


## Issue #1 – My listening streak keeps resetting

### How I reproduced it
I reproduced the bug by running the existing streak tests:

```bash
pytest tests/test_streaks.py
```

The failing test, `test_streak_increments_on_sunday`, simulated a user listening on Saturday followed by Sunday. Instead of increasing the listening streak from 1 to 2, the streak reset to 1.

### How I found the root cause
I started by reading the failing test in `tests/test_streaks.py` to understand the expected behavior. I then traced the call chain:

```
tests/test_streaks.py
    ↓
update_listening_streak()
    ↓
services/streak_service.py
```

Reading the implementation of `update_listening_streak()` revealed a condition that prevented the streak from incrementing whenever `today.weekday() == 6` (Sunday). Since this matched the exact scenario in the failing test, I confirmed it was the root cause.

### The root cause
The streak logic incorrectly excluded Sundays from the consecutive-day increment condition:

```python
elif days_since_last == 1 and today.weekday() != 6:
```

Python's `datetime.weekday()` returns `6` for Sunday. As a result, listening on Saturday followed by Sunday was treated as a broken streak instead of a consecutive day, causing the streak to reset to 1.

### My fix and side-effect check
I removed the unnecessary Sunday check so that any consecutive calendar day increments the streak:

```python
elif days_since_last == 1:
```

After making the change, I verified the fix by running:

```bash
pytest tests/test_streaks.py
pytest
```

The Sunday test now passes, and the full test suite passes successfully. I also confirmed that the existing behaviors for first-time listeners, multiple listens on the same day, and skipped days still work correctly.

### AI usage
After tracing the failing test to `update_listening_streak()`, I used ChatGPT to help explain the streak logic and confirm how `datetime.weekday()` works. ChatGPT helped explain why the Sunday check was unnecessary, but I verified the diagnosis myself by reading the code and confirming it matched the failing test before making the change.