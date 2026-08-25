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
  Publish contextual widgets to the bottom of the user's watchface from an
  Alloy app.
guide_group: alloy
order: 17
---

The `pebble/peek` module lets an Alloy app publish a Quick View widget: one
line of glanceable, contextual information — an icon, a title, and an optional
subtitle — that peeks up from the bottom of the user's watchface after the app
exits, on the same surface Timeline Quick View uses for upcoming events. While
the widget is showing, the user's Down button becomes a shortcut straight back
into your app.

For the full picture of how the surface is shared and what the user can
configure, read the C-focused
[*Quick View Widgets*](/guides/user-interfaces/quick-view-widgets/) guide; the
behavior is identical for Alloy apps.

## Publishing and Withdrawing

An app has at most one widget; publishing again replaces it:

```js
import PeekWidget from "pebble/peek";

const result = PeekWidget.publish({
    title: "Bus 42 leaves soon",
    subtitle: "Elm St in 7 minutes",
    launchCode: 42,
    timeout: 7 * 60,
});

if (result !== "success")
    console.log(`Widget rejected: ${result}`);
```

`publish()` accepts these options:

| Option | Description |
|--------|-------------|
| `title` | Required. Truncated at 32 bytes of UTF-8 on a codepoint boundary. |
| `subtitle` | Optional second line, truncated the same way. |
| `icon` | A `publishedMedia` id from `package.json`; omit for a generic icon. Tiny (25x25) icons fit best. |
| `launchCode` | A number handed back to the app when the widget's button shortcut launches it. |
| `timeout` | Seconds until the widget withdraws itself; omit or `0` to stay until withdrawn. |

It returns `"success"`, `"invalidArgs"` (for example, a missing title), or
`"disabled"` when the user has turned app widgets off in
*Settings > Quick View* — treat that as an answer, not an error.

Withdraw the widget the moment it no longer applies:

```js
PeekWidget.withdraw();
```

A widget also disappears when its timeout elapses, when the user dismisses it
with the Back button, when the app is uninstalled, or when the watch reboots.

## Handling the Button Shortcut

While the widget is on screen, the user's Down button (per their
*Settings > Quick View > Open With Down* choice) launches your app. The launch
reason is `APP_LAUNCH_PEEK_WIDGET` (the value `8` in `watch.launch.reason`)
and `watch.launch.arguments` carries the widget's `launchCode`:

```js
const LAUNCH_REASON_PEEK_WIDGET = 8;

if (watch.launch.reason === LAUNCH_REASON_PEEK_WIDGET) {
    if (watch.launch.arguments === 42)
        showDepartureDetails();
}
```

If the widget asked the user to act, acknowledge it by calling
`PeekWidget.withdraw()` when the matching launch code arrives.

Because widgets outlive the app, they pair naturally with
[wakeups](/guides/alloy/wakeups/): publish the current state with a timeout,
schedule a wakeup for the moment the state changes, and publish the new state
when the wakeup relaunches the app.

> **Platform Support**: Alloy is available on Emery (Pebble Time 2) and Gabbro
> (Pebble Round 2), and `pebble/peek` requires a firmware that includes the
> Quick View widget service. On older firmware the module is unavailable and
> the mod fails to load, so ship widget features in a firmware-matched release.
