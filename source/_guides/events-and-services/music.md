---
# Copyright 2026 Core Devices LLC
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

title: Music
description: |
  Show now-playing information and control phone media from a C app.
guide_group: events-and-services
order: 12
related_docs:
  - MusicService
---

The ``MusicService`` API gives apps and watchfaces access to the same phone
music state used by Pebble's built-in Music app. It can provide the current
track, artist, album, player name, playback state, estimated progress, and
volume. Apps can also send the playback commands supported by the connected
phone.

The watch normalizes two different transports behind this API. iPhones use
Apple Media Service over Bluetooth LE, while Android phones exchange music
state and commands through the Pebble companion app. An app does not need to
know which transport is active, but it must check capabilities because the
available information and commands can differ by phone platform, connection,
track, and companion app version.

> **Platform Support**: The C API is available on Emery (Pebble Time 2), Flint,
> and Gabbro (Pebble Round 2). The current SDK keeps older platforms frozen at
> their released API revisions. Wrap MusicService declarations and calls in
> ``#if PBL_API_EXISTS(music_service_has_now_playing)`` when a project also
> targets an older platform.

> **Permissions**: MusicService does not currently show a per-app permission
> prompt. An installed app that uses this API can read the cached media state
> and send commands supported by the connected phone while the app is running.

Alloy apps use the [`pebble/music`](/guides/alloy/music/) module instead. Both
APIs read and control the same firmware service; no phone-side JavaScript is
required.


## Reading Now-Playing Metadata

First check whether metadata is available, then provide three
``MUSIC_SERVICE_BUFFER_LENGTH``-byte buffers:

```c
static char s_title[MUSIC_SERVICE_BUFFER_LENGTH];
static char s_artist[MUSIC_SERVICE_BUFFER_LENGTH];
static char s_album[MUSIC_SERVICE_BUFFER_LENGTH];

static void update_track_text(void) {
  if (!music_service_has_now_playing()) {
    text_layer_set_text(s_title_layer, "Nothing playing");
    text_layer_set_text(s_artist_layer, "");
    return;
  }

  music_service_get_now_playing(s_title, s_artist, s_album);
  text_layer_set_text(s_title_layer, s_title);
  text_layer_set_text(s_artist_layer, s_artist);
}
```

Each string is null-terminated and may be truncated to fit the watch's
64-byte cache. An unknown field is returned as an empty string. The buffers
must remain valid for as long as a layer uses them.

The active player's name is optional:

```c
char player_name[MUSIC_SERVICE_BUFFER_LENGTH];
if (music_service_get_player_name(player_name)) {
  APP_LOG(APP_LOG_LEVEL_INFO, "Playing from %s", player_name);
}
```

Player names are normally available through the Android music transport. They
may be unavailable on iPhone, where the watch's Apple Media Service client does
not currently request player-name updates.


## Reading Playback State, Progress, and Volume

Call ``music_service_get_playback_info()`` and test its capability bits before
using optional fields:

```c
MusicServicePlaybackInfo info;
music_service_get_playback_info(&info);

if (info.capabilities & MusicServiceCapabilityPlaybackState) {
  bool is_playing = info.playback_state == MusicServicePlaybackStatePlaying;
  // Update a play/pause icon
}

if (info.capabilities & MusicServiceCapabilityProgress) {
  uint8_t percent = info.duration_ms == 0 ? 0 :
      (uint8_t)(((uint64_t)info.position_ms * 100) / info.duration_ms);
  update_progress_bar(percent);  // Application-defined drawing helper
}

if (info.capabilities & MusicServiceCapabilityVolume) {
  APP_LOG(APP_LOG_LEVEL_INFO, "Phone volume: %u%%",
          (unsigned)info.volume_percent);
}
```

``position_ms`` is estimated from the last position received from the phone and
the current playback rate. The phone does not normally send an event every
second, so a visible progress bar should refresh itself with an ``AppTimer``
while the track is playing. A one-second timer is usually smooth enough and
avoids unnecessary redraws:

```c
static void progress_timer_callback(void *context) {
  s_progress_timer = NULL;

  MusicServicePlaybackInfo info;
  music_service_get_playback_info(&info);

  if (info.capabilities & MusicServiceCapabilityProgress) {
    uint8_t percent = info.duration_ms == 0 ? 0 :
        (uint8_t)(((uint64_t)info.position_ms * 100) / info.duration_ms);
    update_progress_bar(percent);
  }

  if ((info.capabilities & MusicServiceCapabilityProgress) &&
      info.playback_state == MusicServicePlaybackStatePlaying) {
    s_progress_timer = app_timer_register(1000, progress_timer_callback, NULL);
  }
}
```

Start the timer only when progress is supported and playback begins, and cancel
it when its window unloads. Watchfaces should also stop high-frequency redraws
while they are out of focus to conserve battery.


## Receiving Music Changes

Subscribe once during app initialization and refresh only the affected UI:

```c
static void music_event_handler(MusicServiceEventType event_type) {
  switch (event_type) {
    case MusicServiceEventNowPlayingChanged:
      update_track_text();
      break;
    case MusicServiceEventPlaybackStateChanged:
    case MusicServiceEventTrackPositionChanged:
    case MusicServiceEventVolumeChanged:
      update_playback_ui();
      break;
    case MusicServiceEventServerConnected:
    case MusicServiceEventServerDisconnected:
      update_track_text();
      update_playback_ui();
      break;
  }
}

static void init(void) {
  // Create windows and layers first
  music_service_subscribe(music_event_handler);
  update_track_text();
  update_playback_ui();
}

static void deinit(void) {
  music_service_unsubscribe();
  // Destroy windows and layers
}
```

The initial read is important: subscribing only reports future changes and
does not immediately invoke the handler with the current state.


## Sending Playback Commands

Always check a command before presenting or sending it. For example, a select
button can toggle playback only when that action is supported:

```c
static void select_click_handler(ClickRecognizerRef recognizer, void *context) {
  const MusicServiceCommand command = MusicServiceCommandTogglePlayPause;
  if (!music_service_is_command_supported(command)) {
    return;
  }

  music_service_send_command(command);
}
```

The command set includes play, pause, play/pause toggle, next and previous
track, volume up and down, skip forward and backward, repeat and shuffle mode,
like, dislike, and bookmark. iOS Apple Media Service may expose more commands
than the Android Pebble Protocol transport. Hide or disable actions that the
connected transport does not support. A player may still ignore a command that
its transport can send.

Commands are delivered on a best-effort basis. A ``true`` return from
``music_service_send_command()`` means the command was supported and handed to
the connected music backend; it is not confirmation that the phone player
performed the action. Observe subsequent music-service events to update the
UI from the phone's reported state.


## Designing a Music Watchface

For a watchface that shows the current track, keep the music area compact and
retain the time as the dominant information. Truncate or scroll long text
deliberately, show a calm empty state when no metadata is available, and avoid
redrawing the whole screen for every progress update. In a dedicated music
watchapp, use familiar Pebble button conventions: Up for previous, Select for
play/pause, and Down for next. The system owns those buttons while a watchface
is active, so playback controls belong in a watchapp rather than a watchface.

The Music C Watchface example shows this pattern, including capability-aware
progress and a low-frequency refresh timer.
