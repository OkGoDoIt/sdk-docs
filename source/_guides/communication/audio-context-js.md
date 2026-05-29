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

title: Audio Context with PebbleKit JS
description: |
  How to query transcripts and receive live or raw audio context on the phone.
guide_group: communication
order: 5
related_docs:
  - Pebble
---

PebbleKit JS is the richest Audio Context runtime because it runs on the phone.
Use it when your app needs transcript history, live transcript subscriptions,
raw audio, local phone storage, or web APIs.

Audio Context still follows the system-owned model: your app observes or
queries data from background audio that the user controls. It does not start or
stop the background recorder.

## Capabilities

Declare the capabilities your phone-side code needs:

```json
"capabilities": [
  "audio_status",
  "audio_transcript",
  "audio_history"
]
```

Add `audio_raw` only if your app truly needs raw audio chunks:

```json
"capabilities": [
  "audio_status",
  "audio_raw"
]
```

The mobile app grants or revokes each audio scope per app UUID.

## Promise and Error Model

Most `Pebble.audioContext` methods return Promises. Rejections use an object
with `availability` and `message`:

```js
Pebble.audioContext.getStatus()
  .then(function(status) {
    console.log('Audio availability: ' + status.availability);
  })
  .catch(function(error) {
    console.log('Audio status failed: ' + error.availability);
    console.log(error.message);
  });
```

Common availability values include `DisabledByUser`, `PermissionDenied`,
`CapabilityNotDeclared`, `TranscriptionUnavailable`, `NoData`, and `Error`.

## Status and Enable Flow

Use status before showing an audio-dependent UI:

```js
Pebble.audioContext.getStatus()
  .then(function(status) {
    if (status.availability === 'DisabledByUser') {
      return Pebble.audioContext.requestEnable();
    }

    return status;
  })
  .then(function(result) {
    console.log('Audio status or prompt result: ' + JSON.stringify(result));
  });
```

`requestEnable()` asks the system to show the user an enable flow. The app does
not directly enable background audio.

## Request Permission

Ask for permission close to the action that needs the data:

```js
Pebble.audioContext.requestPermission(['audio_transcript'])
  .then(function(result) {
    console.log('Permission prompt result: ' + result.result);
  })
  .catch(function(error) {
    console.log('Permission failed: ' + error.availability);
  });
```

If the app did not declare the capability in `package.json`, the request will
not become a useful user prompt. Fix the manifest instead of retrying.

## Trigger Metadata

Trigger metadata helps Quick Launch or accessory-triggered apps choose the
right transcript window:

```js
Pebble.audioContext.getTriggerInfo()
  .then(function(trigger) {
    console.log('Launch reason: ' + trigger.launchReason);
    console.log('Source: ' + trigger.sourceType);
  });
```

Some fields may be `null` when the source is unknown.

## Recent Transcript

Recent transcript returns `{ segments: [...] }`:

```js
Pebble.audioContext.recentTranscript({
  beforeSeconds: 30,
  afterSeconds: 5
}).then(function(result) {
  result.segments.forEach(function(segment) {
    console.log(segment.text);
  });
});
```

Segments include timing and transcript metadata:

| Field | Description |
|-------|-------------|
| `id` | Segment identifier |
| `text` | Transcript text |
| `isFinal` | Whether the transcript is final |
| `startedAtEpochMs` | Segment start time |
| `endedAtEpochMs` | Segment end time |
| `language` | Language when available |
| `provider` | Transcription provider when available |
| `modelUsed` | Model name when available |
| `warnings` | Warnings about gaps or partial data |

Handle empty `segments` as a normal result.

## Transcript History

History queries require `audio_history`:

```js
var now = Date.now();

Pebble.audioContext.transcriptHistory({
  startedAtEpochMs: now - 5 * 60 * 1000,
  endedAtEpochMs: now
}).then(function(result) {
  console.log('Segments: ' + result.segments.length);
});
```

Use bounded windows. Do not request broad history unless your app has a clear
user-facing reason.

## Live Transcript

`onTranscript()` resolves to an unsubscribe function:

```js
var stopTranscript;

Pebble.audioContext.onTranscript({ includePartial: true }, function(segment) {
  console.log(segment.text);
}).then(function(unsubscribe) {
  stopTranscript = unsubscribe;
});

function stopLiveTranscript() {
  if (stopTranscript) {
    stopTranscript();
    stopTranscript = null;
  }
}
```

Call the unsubscribe function when the app no longer needs updates.

## Status Subscription

Use `onStatus()` when the UI needs to react to background audio state changes:

```js
var stopStatus;

Pebble.audioContext.onStatus(function(status) {
  console.log('Audio stream state: ' + status.streamState);
}).then(function(unsubscribe) {
  stopStatus = unsubscribe;
});
```

Avoid polling status rapidly. A short-lived subscription is cheaper and easier
to reason about.

## Raw Audio

Raw audio is delivered on the phone through PebbleKit JS. Apps need the
explicit `audio_raw` capability and a separate user grant.

{% alert important %}
Raw audio may include sensitive sounds nearby. Request `audio_raw` only when
transcripts are not enough, and stop the subscription as soon as your app no
longer needs audio samples.
{% endalert %}

```js
var stopRawAudio;

Pebble.audioContext.onRawAudio({ maxChunkBytes: 16000 }, function(chunk) {
  console.log(chunk.encoding + ' ' + chunk.sampleRateHz + ' Hz');
  console.log('Bytes: ' + chunk.base64.length);
}).then(function(unsubscribe) {
  stopRawAudio = unsubscribe;
});

function stopRaw() {
  if (stopRawAudio) {
    stopRawAudio();
    stopRawAudio = null;
  }
}
```

Raw chunks contain:

| Field | Description |
|-------|-------------|
| `streamId` | Background audio stream identifier |
| `sequenceStart` | Sequence number for this chunk |
| `sampleIndexStart` | First sample index in this chunk |
| `timestampEpochMs` | Chunk timestamp when available |
| `sampleRateHz` | Sample rate |
| `channels` | Channel count |
| `encoding` | Current value is `Pcm16Le` |
| `base64` | Base64-encoded audio bytes |
| `gapCountSinceLastChunk` | Capture gaps since the previous chunk |

Chunks are not browser `AudioBuffer` objects. Decode the base64 data before
passing it to APIs that expect binary audio.

## Cloud and Webhook Work

PebbleKit JS can use `XMLHttpRequest`, `localStorage`, and other phone-side
features. When sending transcript or raw audio to a service:

* Make the upload visible in your app UI or configuration.

* Send only the minimum window needed.

* Prefer transcript text over raw audio.

* Stop live subscriptions when the upload task is done.

* Handle mobile network failures and retry intentionally.

## Compatibility

Audio Context depends on support from the watch firmware and phone app. Always
handle `UnsupportedWatch`, `UnsupportedPhone`, disabled background audio, denied
permission, and missing transcript data.

For watch-side UI examples, see [Audio Context in C][audio-context-c].
For Alloy, see [Audio Context in Alloy][audio-context-alloy].

[audio-context-c]: /guides/events-and-services/audio-context-c/
[audio-context-alloy]: /guides/alloy/audio-context/
