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
  Show now-playing information and control phone media from an Alloy app.
guide_group: alloy
order: 17
---

The `pebble/music` module gives Alloy apps and watchfaces access to the same
phone media state used by Pebble's built-in Music app. It provides the current
track, artist, album, player name, playback state, estimated progress, volume,
and the playback commands supported by the connected phone.

The watch hides the differences between Apple Media Service on iPhone and the
Pebble music protocol on Android. An app does not need phone-side JavaScript,
but it must check capabilities because the available information and commands
can differ by phone platform, connection, and companion app version.

> **Platform Support**: The module is available on Emery (Pebble Time 2) and
> Gabbro (Pebble Round 2), the platforms currently supported by Alloy.

> **Permissions**: `pebble/music` does not currently show a per-app permission
> prompt. An installed app that uses this module can read the cached media state
> and send commands supported by the connected phone while the app is running.

## Creating a Music Service

Import `MusicService` and create one instance. The `onChange` callback is
optional, but subscribing lets the UI react immediately when the phone reports
a change:

```js
import MusicService from "pebble/music";

const music = new MusicService({
    onChange(event) {
        console.log("Music changed: " + event);
        updateMusicDisplay();
    }
});
```

Only one `MusicService` instance can be open at a time. Read the initial state
after creating it: `onChange` reports future changes and is not invoked with an
initial snapshot.

## Reading Now-Playing Metadata

`nowPlaying()` returns the current metadata, or `undefined` when none is
available:

```js
const nowPlaying = music.nowPlaying();

if (nowPlaying) {
    console.log(nowPlaying.title);
    console.log(nowPlaying.artist);
    console.log(nowPlaying.album);
}
```

Unknown fields are empty strings. Each string may be truncated to fit the
watch's 64-byte music cache. The active player name is separate and optional:

```js
const player = music.playerName();
if (player)
    console.log("Playing from " + player);
```

Player names are normally available through the Android music transport. They
may be unavailable on iPhone, where the watch's Apple Media Service client does
not currently request player-name updates.

## Reading Playback State, Progress, and Volume

`playbackInfo()` returns a snapshot with the following properties:

| Property | Description |
|----------|-------------|
| `state` | `"unknown"`, `"playing"`, `"paused"`, `"forwarding"`, or `"rewinding"`. |
| `position` | Estimated position in the track, in milliseconds. |
| `duration` | Track duration in milliseconds. |
| `playbackRate` | Playback rate as a percentage; `100` is normal and `0` is paused. |
| `volume` | Phone player volume from `0` to `100`. |
| `capabilities` | Booleans identifying the optional values reported by the phone. |

Always test the corresponding capability before using an optional value:

```js
const playback = music.playbackInfo();

if (playback.capabilities.playbackState)
    console.log("State: " + playback.state);

if (playback.capabilities.progress && playback.duration > 0) {
    const percent = Math.floor(100 * playback.position / playback.duration);
    console.log("Progress: " + percent + "%");
}

if (playback.capabilities.volume)
    console.log("Volume: " + playback.volume + "%");
```

`position` is estimated from the last position and playback rate reported by
the phone. A visible progress bar can refresh itself with a timer while media
is playing, rather than waiting for a phone event every second:

```js
let progressTimer;

function updateProgressTimer(playback) {
    const playing = playback.capabilities.progress &&
        playback.state === "playing";

    if (playing && progressTimer === undefined) {
        progressTimer = setInterval(updateMusicDisplay, 5000);
    } else if (!playing && progressTimer !== undefined) {
        clearInterval(progressTimer);
        progressTimer = undefined;
    }
}
```

Stop that timer while the watchface is out of focus. A five-second update is
normally smooth enough for a small progress bar without redrawing every
second.

## Receiving Music Changes

The `onChange` callback receives one of these event strings:

| Event | Meaning |
|-------|---------|
| `"nowPlayingChanged"` | The title, artist, album, or player name changed. |
| `"playbackStateChanged"` | The play, pause, forwarding, or rewinding state changed. |
| `"trackPositionChanged"` | The reported track position or duration changed. |
| `"volumeChanged"` | The phone player volume changed. |
| `"serverConnected"` | A phone music service connected. |
| `"serverDisconnected"` | The phone music service disconnected. |

Read a fresh snapshot from the callback instead of trying to apply the event
as a state change. A connection or metadata event can affect several displayed
fields at once.

## Sending Playback Commands

Check every command before presenting or sending it:

```js
const command = "togglePlayPause";

if (music.isCommandSupported(command))
    music.sendCommand(command);
```

The supported command strings are:

| Command | Action |
|---------|--------|
| `"play"`, `"pause"`, `"togglePlayPause"` | Change play/pause state. |
| `"nextTrack"`, `"previousTrack"` | Move through the queue. |
| `"volumeUp"`, `"volumeDown"` | Change player volume. |
| `"skipForward"`, `"skipBackward"` | Seek by the player's standard interval. |
| `"advanceRepeatMode"`, `"advanceShuffleMode"` | Cycle repeat or shuffle mode. |
| `"like"`, `"dislike"`, `"bookmark"` | Perform a player-specific rating or bookmark action. |

An unknown command string throws a `RangeError`. A `true` result from
`sendCommand()` means the command was supported and handed to the connected
music backend; it does not confirm that the phone player performed the action.
Update the UI from subsequent `onChange` events and fresh snapshots.

iOS Apple Media Service may expose more commands than the Android Pebble music
protocol. A player may still ignore a command that its transport can send.
Hide or disable an unsupported action instead of assuming that every phone has
the same controls.

## Cleaning Up

Call `close()` when a screen no longer needs the service:

```js
music.close();
```

Closing releases the event subscription. The instance cannot be used again
after it is closed.

## Designing a Music Watchface

Keep the time as the dominant information and use a compact area for the title,
artist, album, and progress. Truncate long text deliberately and hide the music
area when no metadata is available. The system owns the watchface buttons, so
put playback controls in a dedicated watchapp rather than overriding normal
watchface navigation.

The Music Alloy Watchface example uses `pebble/music` with Poco and refreshes
its progress bar only while media is playing. For the C API, see the
{% guide_link events-and-services/music "MusicService guide" %}.
