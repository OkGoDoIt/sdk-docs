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

title: Audio Context in C
description: |
  How to query recent background audio transcripts from C watchapps.
guide_group: events-and-services
order: 6
related_docs:
  - AudioContext
  - AppLaunchReason
  - Dictation
---

The C Audio Context API gives watchapps lightweight access to background audio
status, recent transcript, transcript history, live transcript updates, and
launch trigger metadata.

It follows the style of other Pebble services: request data, receive callbacks,
and clean up subscriptions when the app is done.

Raw audio is not available to C watchapps in v1. Use PebbleKit JS for raw audio
or other phone-side processing.

## Add Capabilities

Declare the capabilities your app needs in `package.json`:

```json
"capabilities": [
  "audio_status",
  "audio_transcript"
]
```

Use `audio_history` only if the app needs bounded transcript history.
Use `audio_raw` only for a PebbleKit JS component that receives raw phone-side audio.

## Check Cached Status

The cached status is available immediately, but it may be stale or unsupported
before the phone has answered a status request.

```c
AudioContextStatus status;
AudioContextAvailability availability =
    audio_context_get_cached_status(&status);

if (availability == AudioContextAvailabilityAvailable) {
  APP_LOG(APP_LOG_LEVEL_INFO, "Recording: %d", status.recording);
}
```

## Request Fresh Status

Use `audio_context_request_status()` to ask the phone for current status.

```c
static void prv_status_callback(AudioContextAvailability result,
                                const AudioContextStatus *status,
                                void *context) {
  if (result != AudioContextAvailabilityAvailable || !status) {
    APP_LOG(APP_LOG_LEVEL_INFO, "Audio status unavailable: %d", (int)result);
    return;
  }

  APP_LOG(APP_LOG_LEVEL_INFO, "Background audio enabled: %d",
          status->background_audio_enabled);
}

static void prv_request_status(void) {
  if (!audio_context_request_status(prv_status_callback, NULL)) {
    APP_LOG(APP_LOG_LEVEL_ERROR, "Could not request audio status");
  }
}
```

The callback receives an ``AudioContextAvailability`` value even when status is
not available. Handle denied, disabled, unsupported, and no-data cases without
crashing the app.

## Ask the User to Enable Background Audio

Apps cannot enable background audio directly. They can ask the system to show an
enable flow:

```c
if (!audio_context_request_enable()) {
  APP_LOG(APP_LOG_LEVEL_ERROR, "Could not request enable flow");
}
```

This function returns whether the request was sent. It does not report whether
the user enabled background audio. Request status afterwards if your UI needs to
know the result.

## Ask for Audio Permission

Permission requests work the same way: the app asks the system to show the
permission flow, and then checks status or data afterwards.

```c
AudioContextPermission permissions =
    AudioContextPermissionStatus |
    AudioContextPermissionRecentTranscript;

if (!audio_context_request_permission(permissions)) {
  APP_LOG(APP_LOG_LEVEL_ERROR, "Could not request audio permission");
}
```

The app must declare the matching capabilities in `package.json`. If it does
not, later requests may fail with `AudioContextAvailabilityCapabilityNotDeclared`.

## Query Recent Transcript

Recent transcript is the usual watchapp workflow. Request a small time window
around the current time:

```c
static char s_last_transcript[256];

static void prv_transcript_callback(AudioContextAvailability result,
                                    const AudioContextTranscript *transcript,
                                    void *context) {
  if (result != AudioContextAvailabilityAvailable || !transcript ||
      !transcript->text) {
    APP_LOG(APP_LOG_LEVEL_INFO, "No transcript: %d", (int)result);
    return;
  }

  snprintf(s_last_transcript, sizeof(s_last_transcript), "%s",
           transcript->text);
  APP_LOG(APP_LOG_LEVEL_INFO, "Transcript: %s", s_last_transcript);
}

static void prv_request_recent_transcript(void) {
  const uint32_t before_seconds = 30;
  const uint32_t after_seconds = 5;

  if (!audio_context_request_recent_transcript(before_seconds, after_seconds,
                                               prv_transcript_callback, NULL)) {
    APP_LOG(APP_LOG_LEVEL_ERROR, "Could not request transcript");
  }
}
```

`transcript->text` is valid only during the callback. Copy it before returning
if your app needs to display or store it later.

## Query Around a Trigger

For Quick Launch or source-neutral action workflows, inspect trigger metadata
first:

```c
static void prv_handle_launch(void) {
  AudioContextTriggerInfo trigger;

  if (audio_context_get_trigger_info(&trigger)) {
    APP_LOG(APP_LOG_LEVEL_INFO, "Launch reason: %d",
            (int)trigger.launch_reason);
  }

  audio_context_request_recent_transcript(30, 5,
                                          prv_transcript_callback, NULL);
}
```

Trigger metadata is best effort. Your app should still work if the source or
action is unknown.

## Query Transcript History

History queries require `audio_history` and should use bounded windows:

```c
time_t now = time(NULL);
time_t start = now - 5 * 60;

audio_context_request_transcript_history(start, now,
                                         prv_transcript_callback, NULL);
```

Use small windows on the watch. For larger history export or cloud processing,
use PebbleKit JS.

## Subscribe to Live Transcript

Live transcript subscriptions deliver transcript updates while the app is
active:

```c
static void prv_subscribe(void) {
  if (!audio_context_subscribe_transcript(prv_transcript_callback, NULL)) {
    APP_LOG(APP_LOG_LEVEL_ERROR, "Could not subscribe to transcript");
  }
}

static void prv_deinit(void) {
  audio_context_unsubscribe();
}
```

Only one watch-side transcript subscription can be active through this API.
Unsubscribe when the app no longer needs updates.

## Availability Values

| Value | What to do |
|-------|------------|
| `AudioContextAvailabilityAvailable` | Continue |
| `AudioContextAvailabilityUnsupportedWatch` | Hide the feature |
| `AudioContextAvailabilityUnsupportedPhone` | Ask the user to update |
| `AudioContextAvailabilityDisabledByUser` | Offer the enable flow |
| `AudioContextAvailabilityPermissionDenied` | Offer the permission flow |
| `AudioContextAvailabilityCapabilityNotDeclared` | Fix `package.json` |
| `AudioContextAvailabilityTranscriptionUnavailable` | Retry later |
| `AudioContextAvailabilityNoData` | Show an empty state |
| `AudioContextAvailabilityError` | Log and degrade gracefully |

## Limitations

The C API is intentionally lightweight:

* It does not expose raw audio.

* Prompt request functions are fire-and-forget booleans.

* Transcript callbacks provide small transcript objects for watch UI flows.

* Apps should keep query windows small and tolerate missing data.

See [Audio Context with PebbleKit JS][audio-context-js] for phone-side raw
audio, larger history windows, and network integrations.

[audio-context-js]: /guides/communication/audio-context-js/
