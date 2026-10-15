## AI tool usage

I used AI to confirm or deny my assumptions about code bugs. I would trace the bugs from the urls to the functions where the bugs existed and make an assumption about the bug, for example what specific line was causing the bug, and I would ask Claude to confirm or deny my assumption without writing any code.

There was another instance where I could not find what line was causing the bug but I new what file it was in so I gave Claude the bug description and asked what in the file might be causing it. It helped walk me through it but there were certain parts that I thought the AI lacked context so I gave it json from the endpoints which allowed it to tighten the root cause.

## Codebase Map

`models.py` - Defines 7 models: User, Tag, Song, ListeningEvent, Rating, Playlist, Notification. It also defines 3 tables: friendships, song_tags, playlist_entries.

`services/feed_service.py` - Builds the friend activity feeds. It defines 2 functions. get_friends_listening_now returns each friend's most recent song from the last 24 hours, with one entry per friend. 
get_activity_feed returns the latest N listening events from all friends with no time cutoff (default 20). Both are sorted newest first.

`services/notification_service.py` - Creates and reads notifications and handles the actions that trigger them. It defines 5 functions. create_notification makes a notification. add_to_playlist adds a song to a playlist and notifies the person who shared the song, unless they added it themselves. rate_song creates or updates a 1–5 rating. get_notifications lists a user's notifications, with an option to show only unread ones. mark_as_read marks a notification as read.

`services/playlist_service.py` - Creates and looks up playlists. It defines 4 functions. create_playlist makes a new playlist, which is collaborative by default. get_playlist_songs returns a playlist's songs in position order. get_playlist returns a playlist's details without its songs. get_user_playlists lists every playlist a user created.

`services/search_service.py` - Handles song lookup. It defines 2 functions. search_songs does a case-insensitive search on song title and artist, and also joins the song's tags. get_song fetches one song by its ID.

`services/streak_service.py` - Tracks listening streaks. It defines 3 functions. record_listening_event logs a listen and updates the user's streak. update_listening_streak applies the rules: the streak starts at 1, stays the same if you already listened today, goes up by 1 if you listened yesterday, and otherwise resets to 1. get_streak returns a user's current streak.

`Dataflow - How song populates user's feed` - POST /songs/<song_id>/listen with body {"user_id": ...} calls listen() in ./songs.py That calls record_listening_event(user_id, song_id) in ./streak_service which: loads the user creates a ListeningEvent and adds it to the session. Calls update_listening_streak(user, now) at streak_service.py to update listening_streak and last_listened_at. Runs db.session.commit(), which saves the event. This is the step that makes the song available to feeds. Then GET /feed/<user_id>/listening-now calls get_friends_listening_now(user_id) at feed_service. After it loads the user and builds friend_ids from user.friends, using the friendships table.
Queries the ListeningEvent rows where user_id is in friend_ids and listened_at is within the last 24 hours, newest first.
Keeps only the newest event for each friend.
For each kept event, loads the User and the Song and returns {friend, song, listened_at}.

`Pattern I noticed` Almost every module in services, looks up by id, guards for error, those some task then commits it to the session.

## Bug Reproduction

`Issue 1` - I reproduced this bug by running the test_streak_increments_on_sunday test. The test sets the streak to 1 on saturday and on sunday it should increment to two however it starts again at one instead

`Issue 3` - I reproduced this bug by going to the url http://127.0.0.1:5000/songs/search?q=Anthem. The result came back with 3 songs all duplicates with the same information.

`Issue 2` - I reproduced the bug by hitting the link http://127.0.0.1:5000/feed/bf60460c-6f03-4092-88a4-25e7961269e1/listening-now. When I hit the link it said darius's last listen was on oct 12 was at 9:15 pm but that he listened to a song on the 13th at 9:05pm

`Issue 4` - I reproduced the bug by first checking http://127.0.0.1:5000/playlists/bd8f14e2-c848-4bb8-87f5-e405b5345be9/songs then trying to add a song and checking the route again to see if I can see the entry

## Root Cause Analysis

`Issue 1` - I first looked in streak_service.py from there I looked for a function that seemed to be handling the streak updating logic update_listening_streak(user: User, now: datetime) -> None. There I noticed a weird line of code
    elif days_since_last == 1 and today.weekday() != 6:
        user.listening_streak += 1
    else:
        user.listening_streak = 1
This means that on the sixth day being sunday instead of incrementing it hits the else statement. The fix was simply removing and today.weekday() != 6. I checked the users/<id>/streak to make sure the streak was still working

`Issue 3` - I first traced the route /search to the module search_services.py there I found the function search_songs(query: str) -> list[dict] that handled returning songs with the name in the query. Here I found the line .outerjoin(song_tags, Song.id == song_tags.c.song_id). song_tags is a many to many link table so joining Song to it produces one result row per tag. Search_songs is only called by the /songs/search route so there were no side effects.

`Issue 2` - I traced the route from GET /feed/<my_id>/listening-now to feed_service.py. There I saw that the seed hardcodes last_listened_at as now - 1 day and the ListeningEvent seed creates at now - 10 minutes these will always disagree. To fix it I changed RECENT_THRESHOLD from 24hours to 30 minutes. RECENT_THRESHOLD is only used by get_friends_listening_now, which is only called by the /feed/<user_id>/listening-now route. Claude tested a friend who listened 5 minutes ago and also 3 hours ago, a friend who listened 29 minutes ago, and a friend whose only listens were 31 minutes and 20 hours ago. The first two appeared in the feed and the third didn't. 

`Issue 4` - I traced the route from playlist_service.py where I found the the function get_playlist_songs(playlist_id: str) -> list[dict]there I noticed the line
return [song.to_dict() for song in songs[:-1]]
which meant that it excluded the last entry of the list. However I wasn't done searching because I needed to find how a song was added to a playlist.To do so I traced the route in /routes/playlist.py and found the function add_to_playlist(playlist_id: str, song_id: str, added_by_user_id: str) in notification_service.py. Here I found another issue where required columns in playlist_entries are missing meaning the database rejects the entry. To fix this error I removed [:-1] so get_playlist_songs returns every song and directly insert playlist_entries directly to fix POST method when adding song. To do a side effect check tested that the notification logic was unchanged and that the API was working as expected.