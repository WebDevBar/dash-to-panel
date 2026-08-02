# Hide chosen apps from the panel

**Date:** 2026-08-02
**Repo:** `WebDevBar/dash-to-panel` (fork of `home-sweet-gnome/dash-to-panel`)
**Intended as:** a standalone upstream PR, branched from `upstream/master`

<!-- claims-audit: ignore Adw.ActionRow Adw.ComboRow Gtk.SignalListItemFactory should_show() -->
<!-- GTK4/Adwaita/Gio API, not repo symbols. All four verified present against the
     installed introspection on 2026-08-02: Adw.ActionRow, Adw.ComboRow (with an
     enable-search property), Gtk.SignalListItemFactory, Gio.AppInfo.should_show.
     Gio.AppInfo.get_all() returned 145 apps, the first carrying an icon. -->

## Problem

Some windows should never appear in the taskbar. The motivating case is an emoji
picker: a popup summoned with `Super+.`, used for two seconds, then dismissed. It
has no business occupying a taskbar slot, and on GNOME there is no way for the app
itself to say so.

That last point is what makes this a Dash to Panel feature rather than an
application one. `MetaWindow.skip-taskbar` is the flag the panel already honours
(`src/appIcons.js:1856`), but it is **read-only** - verified on Mutter 18:

```
skip-taskbar | flags: 225 | writable: False
```

Mutter computes it and exposes no setter, and `xdg-shell` carries no client request
for it either. So neither the application nor a shell extension can set it. A
Wayland app that wants to stay out of the taskbar has no mechanism at all, and the
only place left to solve it is the component drawing the taskbar.

Scope is the panel only. GNOME's Alt-Tab reads the same read-only property and is a
separate component; excluding an app there is deliberately out of scope.

## Behaviour

A new **Hide from panel** list in the Fine-Tune tab. Apps on it never contribute a
running-window icon to any panel.

A pinned favourite for an excluded app **still appears and still launches**. Only
the running-window instance is suppressed. This keeps two settings from silently
fighting: pinning is a statement about the launcher, excluding is a statement about
running windows, and a user who does both gets a launcher that never sprouts a
running indicator. Filtering the pin away as well would look like the pin had
failed.

Adding an app does **not** modify the favourites list. A setting that quietly
rewrites another setting is worse than one that lets them coexist.

## Identity

Apps are stored by **desktop file id** - `it.mijorus.smile.desktop` - matched at
runtime against `Shell.App.get_id()`.

Chosen over `WM_CLASS` because it is stable across sessions, unaffected by the
Wayland `app_id` inconsistencies that plague class matching, and because it is
precisely what an app picker produces, so the stored value and the displayed value
are the same string.

The limit, stated plainly: a window GNOME cannot map to an installed application
gets a synthesised, window-backed id that is not stable, so such windows cannot be
excluded. This is acceptable - those windows are rare, and no identifier would
serve them reliably.

## Storage

One key, following `context-menu-entries` exactly:

```xml
<key type="s" name="hide-from-panel-apps">
  <default>'[]'</default>
  <summary>Apps whose windows never appear in the panel</summary>
</key>
```

A JSON array of desktop file ids. A single string key keeps the read/write idiom
identical to the neighbouring list setting rather than introducing a second style
of list handling into the same prefs file.

## Filtering

`getInterestingWindows()` in `src/appIcons.js:1854` already filters
`w.skip_taskbar`. The exclusion sits beside it and is keyed on the **app**, not the
window - which is what lets the pinned launcher survive while its windows vanish.

**The `app` argument can be null**, and that path must be handled or the feature
silently fails. `src/taskbar.js:1182` calls `getInterestingWindows(null, ...)`, so
an exclusion that reads `app.get_id()` alone would let excluded windows straight
through there. Each window is resolved through
`Shell.WindowTracker.get_default().get_window_app(w)` when `app` is null, and the
resulting app id is tested against the set.

**`get_window_app()` can return null**, and calling `get_id()` on that would throw
during a split-app redisplay. The surrounding code already guards it - `if (app &&
...)` at `src/taskbar.js:1169`. An unresolved window is **kept**, never filtered:
it has no stable identity, so it can never have been added to the list, and
discarding it would remove a window the user never excluded. This is the same
limitation already stated under Identity, reached from the other direction.

The set is parsed once and cached. **Invalidating the cache is not enough on its
own** - it does not rebuild anything, so existing icons would linger until some
unrelated app or window event triggered a redisplay. `changed::hide-from-panel-apps`
must therefore both invalidate the cache and queue a redisplay on every panel,
alongside the existing `changed::` handlers in `src/taskbar.js` (see the block at
lines 388-411, which already pairs setting changes with `_queueRedisplay`).

## Preferences UI

Fine-Tune tab, a new group with an add button in the header - the same arrangement
as the context-menu list in `SettingsAction.ui`, so the tab reads consistently.

**Each configured app is an `Adw.ActionRow`** carrying its icon, its display name,
and a remove button (`user-trash-symbolic`, circular, frameless).

`prefs.js` already has a `createButton` helper for exactly this, but it is a `let`
inside `_bindSettings()` at line 2993, not a module-level function. Fine-Tune
bindings live in the same method (`animate-appicon-hover` is bound at line 3547),
so the new code can reuse it **only if it is placed after line 2993**. Placed
before, it must define its own. Removal is per row and immediate.

Deliberately **not** an `Adw.ExpanderRow` with up/down arrows, which is what the
context-menu list uses. Order is meaningless for a set, and an expander would hide
the single fact the row exists to show.

**The picker is an `Adw.ComboRow` with `enable-search`**, populated from
`Gio.AppInfo.get_all()` filtered through `should_show()`, so it offers what the app
grid offers.

**Every entry in that picker shows the app's icon beside its name**, and so does
the selected value. Names alone are ambiguous - several installed apps can read
"Emoji Picker", and flatpak ids are not shown to the user - so the icon is what
confirms at a glance that the right app was picked. Implemented with a
`Gtk.SignalListItemFactory` binding a `Gtk.Image` from the app's `GIcon` plus a
`Gtk.Label`, applied to both the list and the selected-item display.

Apps already on the list are omitted from the picker, so the same app cannot be
added twice.

## Error handling

- Malformed JSON in the setting yields an empty list rather than throwing, so a
  hand-edited value cannot break the panel.
- A stored id whose application has since been uninstalled stays in the setting and
  is shown with a fallback icon and its raw id, so the user can see and remove it
  rather than finding a blank row.
- An empty list is the default and costs one `length` check on the filter path.

## Testing

- Unit-level: the filter helper, given a list and an app id, includes and excludes
  correctly; malformed JSON yields an empty list.
- Manual, on GNOME 50 Wayland with panels on two monitors:
  1. Add Smile. Open it with `Super+.`. It must not appear on either panel, and
     auto-paste must still work.
  2. Pin Smile as a favourite. The launcher appears; opening it adds no running
     indicator and no second icon.
  3. Remove Smile from the list **while it is open**. Its icon reappears without a
     shell restart and without touching another window - this is what proves the
     redisplay is queued rather than merely the cache cleared.
  5. Repeat step 1 in split-app mode, the only path calling
     `getInterestingWindows(null, ...)`. `allowSplitApps` is
     `usingLaunchers || (!isGroupApps && !showFavorites)` (`src/taskbar.js:434`),
     where `usingLaunchers` is `!isGroupApps && group-apps-use-launchers`
     (`src/taskbar.js:427`) - so turn `group-apps` **off**, then either turn
     `group-apps-use-launchers` **on** or `show-favorites` **off**. That path
     already resolves windows with `tracker.get_window_app(w)` a few lines below
     the call, so the exclusion should follow the same idiom rather than invent one.
  4. Confirm the picker shows icons, that search filters by name, and that an
     app already added is absent from the picker.
- The panel must not need a relogin for a list change to take effect.

## Out of scope

- Alt-Tab and the overview window list.
- Matching by `WM_CLASS`, title, or wildcard.
- Hiding pinned favourites.
- Per-monitor or per-workspace exclusions.
