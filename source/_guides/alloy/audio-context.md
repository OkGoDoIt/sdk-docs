---
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

title: Audio Context
description: |
  Use system background audio context from Alloy apps.
guide_group: alloy
order: 5
---

Alloy apps can use the `pebble/audio-context` module to query background audio
status and transcript data from watch-side JavaScript.

Audio Context is system-owned. Your Alloy app can request permissioned context,
but it cannot start or stop the background recorder directly.

## Import the Module

```js
import AudioContext from "pebble/audio-context";
```

Create one instance for the part of your app that needs audio context:

```js
const audio = new AudioContext({
  onStatus(status) {
    trace(`audio availability ${status.availability}\n`);
  },
  onTranscript(transcript) {
    trace(`audio transcript ${transcript.text}\n`);
  },
  onError(error) {
    trace(`audio error ${error}\n`);
  }
});
```

`onError` receives a numeric `AudioContextAvailability` value from the watch
runtime.

## Add Capabilities

Declare the matching capabilities in `package.json`:

```json
"capabilities": [
  "audio_status",
  "audio_transcript"
]
```

Use `audio_history` only for bounded history queries. Raw audio is not delivered
through the Alloy watch module.

## Permission Constants

`requestPermission()` takes a bitmask:

```js
AudioContext.Permission.status
AudioContext.Permission.recentTranscript
AudioContext.Permission.transcriptHistory
AudioContext.Permission.liveTranscript
AudioContext.Permission.rawAudio
```

For example:

```js
audio.requestPermission(
  AudioContext.Permission.status |
  AudioContext.Permission.recentTranscript
);
```

The app must still declare the matching capabilities. Permission requests ask
the system to show the user a flow; they do not grant access directly.

## Status

Read the cached status immediately:

```js
const status = audio.getCachedStatus();
trace(`cached availability ${status.availability}\n`);
```

Request fresh status from the phone:

```js
audio.requestStatus();
```

The `onStatus` callback receives:

| Field | Description |
|-------|-------------|
| `availability` | Numeric availability value |
| `backgroundAudioEnabled` | Whether background audio is enabled |
| `recording` | Whether the phone is receiving audio |
| `transcribing` | Whether transcription is available |
| `flags` | Reserved flags |

## Recent Transcript

Request recent transcript with a small window:

```js
audio.requestRecentTranscript({
  beforeSeconds: 30,
  afterSeconds: 0
});
```

`onTranscript` receives:

| Field | Description |
|-------|-------------|
| `text` | Transcript text |
| `startTime` | Segment start time in seconds |
| `endTime` | Segment end time in seconds |
| `gapCount` | Known capture gaps |
| `flags` | Reserved flags |

## Transcript History

History queries require `audio_history`:

```js
const now = Math.floor(Date.now() / 1000);

audio.requestTranscriptHistory({
  startTime: now - 300,
  endTime: now
});
```

Use bounded windows. For larger history exports or network processing, use a
PebbleKit JS component.

## Live Transcript

Subscribe while the app is active:

```js
audio.requestPermission(AudioContext.Permission.liveTranscript);
audio.subscribeTranscript();
```

Stop the subscription when the screen or app no longer needs updates:

```js
audio.unsubscribe();
```

## Cleanup

Call `close()` when the instance is no longer needed:

```js
audio.unsubscribe();
audio.close();
```

This releases the native host object and cancels any active transcript
subscription.

## Limitations

The Alloy module is a lightweight watch-side API:

* It does not expose raw audio chunks.

* It reports errors as numeric availability values.

* It is intended for watch UI flows, not bulk transcript export.

* It should use short query windows and short-lived subscriptions.

Use [Audio Context with PebbleKit JS][audio-context-js] for phone-side raw
audio, cloud services, and larger history processing.

[audio-context-js]: /guides/communication/audio-context-js/
