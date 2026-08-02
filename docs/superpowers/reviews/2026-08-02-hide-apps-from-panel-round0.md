# Round 0 trace - hide-apps-from-panel plan

**Document:** `docs/superpowers/plans/2026-08-02-hide-apps-from-panel.md`
**Date:** 2026-08-02
**Repo state:** `webdevbar-local`, plan committed

Parts 1 and 3 done (rulesets re-read; `claims-audit` clean). This is part 2.

| Cited | What the plan claims | Confirmed | Where |
|---|---|---|---|
| `upstream/master` = `1c0c1f1` | branch point | yes | `git log -1 upstream/master` after fetch |
| `schemas/…gschema.xml:750` | `context-menu-entries` is the `type="s"` JSON-list pattern to copy | yes, key runs 750 to ~770 | read |
| `src/appIcons.js:1854` | `getInterestingWindows` signature and the `skip_taskbar` filter | yes | read 1854-1857 |
| `src/appIcons.js:44-53` | `SETTINGS` and `tracker` already imported, no new imports needed | yes - both in the `from './extension.js'` block | read |
| `JSON.parse` precedent in appIcons | the file already parses a JSON setting | yes - `src/appIcons.js:2296` parses `context-menu-entries` | grep |
| `src/taskbar.js` ~385 | the `changed::` handler block exists | yes, 384-414 | read |
| `changed::show-favorites` calls `this._redisplay()` | the idiom the new handler copies | yes | read 386-397 |
| `src/prefs.js:2993` | `createButton` is a `let` inside `_bindSettings()` | yes; `_bindSettings()` spans 796-3932 | grep + awk |
| `src/prefs.js:23-30` | `Adw`, `Gio`, `GioUnix`, `GObject`, `Gtk` all imported | yes, lines 23, 25, 26, 28, 29 | read |
| prefs settings accessor | `this._settings`, NOT a `SETTINGS` global | yes - `this._settings.set_string` at 654, 804, 831, 840 | grep. **This corrected the plan's first draft.** |
| `Gio.DesktopAppInfo` | deprecated in favour of `GioUnix.DesktopAppInfo` | yes - PyGIDeprecationWarning on use; `prefs.js` already imports `GioUnix` (line 26) | executed. **Also corrected the draft.** |
| `ui/SettingsAction.ui:177-198` | group + circular add button markup to copy | yes | read |
| `ui/SettingsFineTune.ui` | ends with an `AdwPreferencesGroup` inside a page | yes, 4 groups | read tail |
| `package.json` | `npm run lint` = prettier + eslint, and there is no test runner | yes; no `test`/`tests` directory exists | read |
| `Gio.ListStore(item_type=Gio.AppInfo)` | can hold `Gio.AppInfo` items | yes - constructed and appended a real app | executed |
| `Adw.AlertDialog`, `Adw.ResponseAppearance.SUGGESTED`, `Gtk.ClosureExpression` | exist | yes | executed against installed introspection |
| `gtk4-builder-tool` | plan told the engineer to run it | **NO - not installed.** Provided by `gtk4-devel`. Plan corrected to offer install or an `xmllint` fallback | `command -v` |

## Not confirmed - the reviewer should attack these

- **The filter expression in Task 2 Step 2 has never been executed.** Its
  ternary-inside-a-filter shape is the single most error-prone line in the plan,
  and `hidden === null` versus a falsy `false` is exactly where it would go wrong.
- **`Gtk.ClosureExpression.new(GObject.TYPE_STRING, fn, null)` from GJS.** Verified
  the class exists; NOT verified that this argument shape is what GJS accepts, nor
  that `Adw.ComboRow:expression` accepts it.
- **`Adw.AlertDialog.present(widget.get_root())`** - the GTK4 dialog presentation
  idiom changed across libadwaita versions. Not verified against this one.
- **Whether `hideAppsGroup.add(row)` / `.remove(row)` is correct for
  `AdwPreferencesGroup`**, versus the `add_row` used for `AdwExpanderRow` in the
  context-menu code.
