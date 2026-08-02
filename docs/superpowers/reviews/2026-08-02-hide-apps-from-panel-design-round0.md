# Round 0 trace - hide-apps-from-panel design

**Document:** `docs/superpowers/specs/2026-08-02-hide-apps-from-panel-design.md`
**Date:** 2026-08-02
**Repo state:** `webdevbar-local` @ `1e5a5a7`

Part 1 (re-read the five rulesets) and part 3 (`claims-audit`, clean) done. This is
part 2: every file the spec cites, opened in its CURRENT shape.

| Cited | What the spec claims | Confirmed | Where |
|---|---|---|---|
| `src/appIcons.js:1854` | `getInterestingWindows()` is where the taskbar filter lives | yes - `export function getInterestingWindows(app, monitor, isolateMonitors)` | read lines 1854-1857 |
| `src/appIcons.js:1856` | it already filters `w.skip_taskbar` | yes - `.filter((w) => !w.skip_taskbar)` | same read |
| `getInterestingWindows` signature | first arg is a `Shell.App`, may be null | yes - `(app ? app.get_windows() : Utils.getAllMetaWindows())` | same read |
| `MetaWindow.skip-taskbar` | read-only, so no client or extension can set it | yes - `flags: 225, writable: False` | `GObject.list_properties(Meta.Window)` via `GI_TYPELIB_PATH=/usr/lib64/mutter-18`, Meta-18 |
| `src/prefs.js:2993` | `createButton` helper exists | yes - `let createButton = (icon, tooltip_text, clicked) => {` | grep + read |
| `createButton` scope | a `let` inside `_bindSettings()`, not module-level | yes - `_bindSettings()` opens at line 796, next method `_setPreviewTitlePosition()` at 3932, so 2993 is inside it | awk scan for method boundaries |
| `src/prefs.js:3547` | Fine-Tune bindings share that scope | yes - `animate-appicon-hover` bound at 3547, inside the same method | grep |
| `ui/SettingsAction.ui` | carries the add-button + group pattern to copy | yes - `AdwPreferencesGroup id="context_menu_group"` line 178, `GtkButton id="context_menu_add_button"` line 181 | read |
| `context-menu-entries` | an existing JSON-in-a-string list setting | yes - `<key type="s" name="context-menu-entries">` | `schemas/org.gnome.shell.extensions.dash-to-panel.gschema.xml:750` |
| `ui/SettingsFineTune.ui` | the tab exists and holds groups | yes - loaded at `src/prefs.js:206`, contains 4 `AdwPreferencesGroup` | grep |
| `Shell.App.get_id()` | used for app identity | yes, and already used in this repo | `src/utils.js:818,820,903` |
| `Adw.ActionRow` | exists | yes | `GObject` import against installed Adw 1 |
| `Adw.ComboRow` + `enable-search` | exists, has that property | yes - property present in `list_properties()` | same |
| `Gtk.SignalListItemFactory` | exists | yes | against installed Gtk 4.0 |
| `Gio.AppInfo.should_show` | exists | yes | `hasattr(Gio.AppInfo, "should_show")` |
| `Gio.AppInfo.get_all()` | enumerates installed apps carrying icons | yes - 145 apps, first has an icon | executed |

## Not confirmed, and the spec should not be read as claiming it

- **That a window-backed app's id is unstable.** The spec says such windows cannot
  be excluded. I did not construct one to measure it. It is stated as a limitation
  rather than a mechanism, and nothing in the design depends on the claim being
  precise.
- **That filtering by app inside `getInterestingWindows()` leaves a pinned
  favourite visible.** This follows from the pin being a launcher rather than a
  window, but it is reasoning, not a measurement. It is testing step 2 in the spec
  for exactly that reason, and it is the first thing a reviewer should attack.
