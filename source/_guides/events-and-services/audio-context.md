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

title: Audio Context
description: |
  How to use system-owned background audio context in watchapps.
guide_group: events-and-services
order: 5
related_docs:
  - AudioContext
  - Dictation
---

Audio Context lets apps use transcripts and other context derived from the
system background audio stream. It is designed for apps that need to know what
was recently said, why the app was launched, or what transcript is arriving
while the app is open.

The important model is:

* Background audio is a system feature controlled by the user.

* Apps can query or observe a permissioned view of that system data.

* Apps cannot start or stop the background recorder directly.

This keeps capture, transcription, storage, and battery policy in one system
service while still letting apps build useful voice workflows.

## Audio Context and Dictation

Audio Context is not a replacement for ``Dictation``.

Use ``Dictation`` when your app wants a foreground microphone session. The app
starts the session, the user speaks into the dictation UI, and the app receives
the result of that session.

Use Audio Context when your app wants context from the system background audio
stream. The app may have been launched from Quick Launch, a button, or a
source-neutral accessory action. After launch, it can ask for recent transcript
around that moment.

| Need | Use |
|------|-----|
| Foreground user speech input | ``Dictation`` |
| Recent transcript around app launch | Audio Context |
| Live transcript while app is open | Audio Context |
| Raw phone-side audio chunks | Audio Context in PebbleKit JS |

## Choosing a Runtime

Audio Context is available in the Pebble development styles that need it, but
each runtime has a different job.

| Runtime | Best for |
|---------|----------|
| C | Watch UI, status checks, recent transcript, trigger info |
| PebbleKit JS | Larger transcript windows, raw audio, network/cloud work |
| Alloy | Watch-side JavaScript UI and lightweight transcript access |

C and Alloy are watch-side APIs. They are intentionally small and should not be
used to move raw audio to the watch. PebbleKit JS runs on the phone, so it is
the right place for raw audio, history queries, uploads, and web APIs.

## Declaring Capabilities

Audio Context access starts with `package.json`. Declare only the narrowest
capability your app needs:

```json
"capabilities": [
  "audio_status",
  "audio_transcript"
]
```

Available capabilities:

| Capability | Allows |
|------------|--------|
| `audio_status` | Check availability and request the enable flow |
| `audio_transcript` | Read recent transcript and live transcript updates |
| `audio_history` | Read bounded transcript history |
| `audio_raw` | Receive phone-side raw audio chunks |

Declaring a capability does not grant access. The user grants or revokes audio
access per app in the Pebble mobile app.

## Requesting Permission

Apps can ask the system to show an enable or permission flow, but the app does
not control the result. If the user denies or later revokes access, the app must
handle that gracefully.

The safe flow is:

1. Declare the required capability.
2. Check status or request the data your app needs.
3. If permission is missing, ask the system to show the permission flow.
4. Try the status or data request again.
5. Show an empty or unavailable state if access is still missing.

Do not repeatedly ask for audio permission without a clear user action.

## Recent Context

Recent context is the default Audio Context workflow. An app launches normally,
checks why it was launched, and asks for transcript around that time.

For example, a note app can launch from Quick Launch and request the last 30
seconds of transcript. If transcript is not available yet, it can show an empty
state or ask the user to try again later.

Transcript data can be delayed, partial, or missing. Background audio may have
been disabled, the phone may have been disconnected, or transcription may still
be running.

## Trigger Metadata

Audio Context includes source-neutral trigger metadata. Apps should use this
metadata to understand the action that opened them without hard-coding one
piece of hardware.

Trigger metadata can include:

* Launch reason
* Trigger timestamp
* Source type, such as watch, phone, ring, shortcut, or system
* Source action
* Launch button
* Launch arguments

The metadata is best effort. Apps should still work when some fields are
unknown.

## Live Subscriptions

Apps can subscribe to status or transcript updates while active. This is useful
for interfaces that show live transcript as the user works.

Subscriptions should be short-lived. Stop them when the app closes, when a
screen no longer needs updates, or when a phone-side task is complete.

## Raw Audio

Raw audio is available through PebbleKit JS on the phone.
It requires the `audio_raw` capability and a separate user grant.

{% alert important %}
Raw audio may include sensitive sounds nearby. Request `audio_raw` only when
transcripts are not enough, and stop the subscription as soon as your app no
longer needs audio samples.
{% endalert %}

Most apps should use transcripts instead of raw audio. Raw audio is intended for
advanced integrations such as custom transcription, acoustic event detection, or
research workflows that genuinely need samples.

## Availability Values

Every Audio Context runtime reports availability or failure reasons. Apps should
handle each one:

| Availability | Meaning |
|--------------|---------|
| `Available` | Audio Context can be used |
| `UnsupportedWatch` | The watch or firmware does not support Audio Context |
| `UnsupportedPhone` | The phone app does not support Audio Context |
| `DisabledByUser` | The user disabled background audio |
| `PermissionDenied` | The user has not granted the requested scope |
| `CapabilityNotDeclared` | The app did not declare the capability |
| `TranscriptionUnavailable` | Transcript service is unavailable |
| `NoData` | No transcript exists for the requested window |
| `Error` | An unexpected failure occurred |

Treat `NoData` as a normal empty state, not as a crash-worthy error.

## Privacy and Battery

Audio Context is designed to avoid duplicating capture work. The system captures
once, transcribes once, stores once, and fans out permissioned views to apps.

Apps should still be careful:

* Prefer transcript over raw audio.

* Prefer recent queries over long live subscriptions.

* Unsubscribe when the app no longer needs updates.

* Explain uploads or cloud processing in your app UI or configuration.

* Expect gaps, delays, disabled background audio, and revoked permission.

## Next Steps

Use the runtime guide that matches your app:

* {% guide_link events-and-services/audio-context-c "Audio Context in C" %}

* [Audio Context with PebbleKit JS][audio-context-js]

* {% guide_link alloy/audio-context "Audio Context in Alloy" %}

[audio-context-js]: /guides/communication/audio-context-js/
