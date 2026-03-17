# KeyCastr Key Filter

Fork KeyCastr to add per-key filtering so only navigation and modifier keys are displayed.

## Problem

When demoing keyboard-driven workflows (e.g. tab navigation, accessibility testing), you want
an on-screen keystroke overlay — but existing tools like KeyCastr show *every* key. Typing
regular text floods the overlay and distracts from the keys that actually matter. KeyCastr's
preferences don't support filtering by key type.

Other macOS keystroke visualizers (Keystrokes, etc.) have the same limitation. Linux tools like
ShowMeTheKey and Windows tools like Carnac offer some filtering, but nothing on macOS does.

## Idea

Fork [KeyCastr](https://github.com/keycastr/keycastr) (Objective-C, open source) and add a
key filter option to preferences. The filter would let users choose which keys to display:

- **Navigation keys** — Tab, Space, Arrow keys, Enter, Escape
- **Modified keys** — any key combined with Cmd, Ctrl, Alt, or Shift (e.g. Shift+Tab, Cmd+A)
- **Regular character keys** — suppressed by default in this mode

This covers the common demo scenario: showing *how* you're navigating (Tab, Shift+Tab, arrows,
Space to activate) without cluttering the display with incidental typing.

## Implementation

The keystroke display logic likely lives in the visualizer classes. The change would be a
filter check before displaying each keystroke:

1. If the event has modifier flags (Cmd/Ctrl/Alt/Shift), always show it.
2. If the keycode matches Tab, Space, arrows, Enter, or Escape, always show it.
3. Otherwise, suppress the display.

This could be exposed as a checkbox in preferences ("Show navigation and modifier keys only")
or as a more granular key picker.

## Custom modes instead of a hardcoded filter

Rather than adding a single "navigation keys only" mode, it might be better to add support for
user-defined custom display modes in KeyCastr. Users could create their own filter
configurations specifying which keys to show/hide, and switch between them from the menu or
preferences. The navigation-keys-only filter described above would just be one example
configuration. This would make the feature more generally useful and more likely to be accepted
upstream.

## Contribution

This could be useful to the broader KeyCastr community — worth proposing upstream as a PR
rather than maintaining a private fork long-term.
