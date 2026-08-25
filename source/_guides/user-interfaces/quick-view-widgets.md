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

title: Quick View Widgets
description: |
  How to publish contextual widgets to the bottom of the user's watchface.
guide_group: user-interfaces
order: 7
related_docs:
  - PeekWidget
  - LaunchReason
  - UnobstructedArea
---

Timeline Quick View has always been able to peek up from the bottom of the
watchface to show an upcoming event. Quick View widgets open that same surface
to apps: one line of glanceable, contextual information — an icon, a title,
and an optional subtitle — that stays visible on the watchface after your app
exits. While a widget is showing, the Down button becomes a shortcut straight
back into the app that published it.

The surface is deliberately shared and user-controlled. The system arbitrates
between everything that wants to peek — an upcoming timeline event or a
recently arrived notification will show above an app's widget — and the user
can dismiss any widget with the Back button, or disable app widgets entirely
in *Settings > Quick View*.

> The ``PeekWidget`` API is only applicable to watchapps and their workers, it
is not supported by watchfaces.

## Publishing a Widget

An app has at most one widget, described by a ``PeekWidgetInfo``. Publishing
again replaces it:

```c
static void prv_publish_departure_widget(void) {
  const PeekWidgetInfo info = (PeekWidgetInfo) {
    .icon = PUBLISHED_ID_BUS,
    .title = "Bus 42 leaves soon",
    .subtitle = "Elm St in 7 minutes",
    .launch_code = 42,
    .timeout_s = 7 * SECONDS_PER_MINUTE,
  };

  const PeekWidgetResult result = peek_widget_publish(&info);
  if (result != PEEK_WIDGET_RESULT_SUCCESS) {
    APP_LOG(APP_LOG_LEVEL_ERROR, "Widget rejected (%d)", (int)result);
  }
}
```

The `title` is required; both strings are copied during the call and truncated
at ``PEEK_WIDGET_TITLE_MAX_LEN`` and ``PEEK_WIDGET_SUBTITLE_MAX_LEN`` bytes of
UTF-8 on a codepoint boundary. ``peek_widget_publish()`` returns
``PEEK_WIDGET_RESULT_DISABLED`` when the user has turned app widgets off —
treat that as an answer, not an error.

The `icon` is a `publishedMedia` identifier from your `package.json`, exactly
as used by ``AppGlanceSlice``. Tiny (25x25) icons fit the widget's icon box
best. Pass `0` to use a generic icon.

## Widget Lifetime

A widget is contextual, not persistent. It disappears when the first of these
happens:

* Its `timeout_s` elapses. `0` means no timeout, capped only by the points
  below.
* The app calls ``peek_widget_withdraw()``.
* The user dismisses it with the Back button.
* The app is uninstalled, or the watch reboots.

Publish a widget when your app has something timely to say — a timer nearing
zero, a departure coming up, a score change — give it an honest timeout, and
withdraw it the moment it no longer applies. A widget that is always present
is a widget the user will disable.

Because widgets survive app exit, a typical pattern pairs the widget with the
``Wakeup`` API: publish "steeping" with a timeout, schedule a wakeup for the
moment the state changes, and publish the new state when the wakeup relaunches
the app.

## The Button Shortcut and Launch Codes

While a widget is on screen, the user's Down button launches the publishing
app instead of its normal assignment. The gesture is chosen by the user in
*Settings > Quick View > Open With Down*: a single press (the default), a
double press, a hold, or disabled entirely. Your app cannot observe or change
that choice — it simply gets launched.

Such a launch reports ``APP_LAUNCH_PEEK_WIDGET`` from ``launch_reason()``, and
``launch_get_args()`` returns the widget's `launch_code`, so the app can jump
straight to whatever the widget was about:

```c
static void prv_init(void) {
  if (launch_reason() == APP_LAUNCH_PEEK_WIDGET) {
    switch (launch_get_args()) {
      case LAUNCH_CODE_DEPARTURE:
        prv_show_departure_details();
        break;
      default:
        break;
    }
  }
  // ...
}
```

The launch code is also how an app acknowledges a widget: if the widget asked
the user to act ("Tea is ready!"), call ``peek_widget_withdraw()`` when the
matching launch code arrives.

## Sharing the Surface

Only one thing peeks at a time. The system shows, in priority order: a
recently arrived notification, Timeline Quick View, the freshest app widget,
and the now-playing music widget. Your widget appears when nothing above it is
active, and reappears when a higher-priority peek goes away — there is no
queue to manage and no notification when arbitration changes.

Widgets render with the same geometry as Timeline Quick View, so watchfaces
that already handle the ``UnobstructedArea`` API adapt to them without any
changes. If you are building a watchface, see the
[*Unobstructed Area*](/guides/user-interfaces/unobstructed-area/) guide — a
widget is just another reason the obstruction can appear.

## User Settings

Everything about the surface is under the user's control in
*Settings > Quick View*:

* **Music** — the built-in now-playing widget: Disabled, While Playing, or
  only at the Start of Track.
* **Notifications** — how long a new notification lingers as a peek after the
  notification popup is gone: Disabled, 5 Seconds, 15 Seconds, 1 Minute, or
  Until Dismissed.
* **Show When Muted** — by default the notification widget follows the popup's
  gating, so muted notifications never peek. Turning this on keeps the widget
  appearing even when the notification filter or Quiet Time has silenced the
  popups — it shows immediately, with no vibration and no backlight, as a calm
  indicator for users who keep notification popups off entirely.
* **App Widgets** — the master toggle for third-party widgets.
* **Open With Down** — the button shortcut gesture.

Design your app so it remains fully usable when the user has turned app
widgets off; the widget should be a shortcut, never the only path.

> **Platform Support**: The C API is available on Emery (Pebble Time 2),
> Flint, and Gabbro (Pebble Round 2). The current SDK keeps older platforms
> frozen at their released API revisions; on those platforms
> ``peek_widget_publish()`` compiles to a stub that returns
> ``PEEK_WIDGET_RESULT_DISABLED``. Wrap calls in
> ``#if PBL_API_EXISTS(peek_widget_publish)`` when a project also targets an
> older platform.

Alloy apps use the [`pebble/peek`](/guides/alloy/quick-view-widgets/) module
instead. Both APIs publish through the same firmware service.
