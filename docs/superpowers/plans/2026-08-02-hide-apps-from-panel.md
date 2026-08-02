# Hide Apps From Panel Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Let the user name apps whose windows never appear in the Dash to Panel taskbar, configured from a searchable app picker in the Fine-Tune tab.

**Architecture:** One JSON-in-a-string GSetting holds desktop file ids. `getInterestingWindows()` in `src/appIcons.js` drops windows whose resolved `Shell.App` id is in that set, which suppresses running-window icons while leaving pinned favourites untouched. `src/taskbar.js` queues a redisplay when the setting changes. The prefs UI copies the existing context-menu-entries pattern.

**Tech Stack:** GJS (ES modules), GTK 4, libadwaita 1, GSettings/GSchema, GNOME Shell 46-50.

**Spec:** `docs/superpowers/specs/2026-08-02-hide-apps-from-panel-design.md` — five rounds of peer review, all findings applied.

## Global Constraints

<!-- claims-audit: creates /tmp/add-key.py /tmp/hidden-check.js /tmp/pr-body.md -->
<!-- Scratch files the plan tells the engineer to write. Deliverables of their
     own steps, not citations of anything expected to exist beforehand. -->

- Branch from `upstream/master` (`1c0c1f1`), **code only**. Do not include `docs/` in this branch; the spec and this plan live on `webdevbar-local`.
- Target branch name: `feat/hide-apps-from-panel`.
- No unit-test framework exists in this repo. `npm run lint` (prettier + eslint) is the only automated gate; correctness is proven by the manual GNOME steps in Task 5.
- K&R braces, no semicolons, single quotes — enforced by prettier, so run the linter rather than hand-formatting.
- Never use the Edit tool on `.js`, `.xml` or `.ui` files — it silently converts straight quotes to curly ones and corrupts them.
- **Every edit below is an anchored Python replace with `assert s.count(anchor) == 1`.** Unified diffs are the usual default, but line numbers shift as these tasks apply in sequence, so a diff written now would not apply cleanly by Task 4. The assert is what makes a failed match loud instead of a silent no-op — never remove it.
- The setting key is `hide-from-panel-apps` everywhere. Never `hidden-apps` or `hide-apps`.
- `Adw.AlertDialog` requires **libadwaita 1.5**, which is exactly what GNOME 46 ships — the floor in `metadata.json` (`shell-version: 46-50`). It is available across the whole supported range, but only just, so do not lower that floor without replacing the dialog.
- All user-facing strings must be wrapped in `_()` in `.js` and marked `translatable="yes"` in `.ui`.
- Requires a **full GNOME relogin** to test any `src/*.js` change — GNOME caches extension ES modules for the life of the shell process. Editing and re-enabling the extension is NOT sufficient and will show stale behaviour.

---

## File Structure

| File | Responsibility | Action |
|---|---|---|
| `schemas/org.gnome.shell.extensions.dash-to-panel.gschema.xml` | declares `hide-from-panel-apps` | modify |
| `src/appIcons.js` | the exclusion helper + the filter hook | modify |
| `src/taskbar.js` | queue a redisplay when the list changes | modify |
| `ui/SettingsFineTune.ui` | the group and its add button | modify |
| `src/prefs.js` | row construction, app picker, add/remove | modify |

---

### Task 1: Schema key

**Files:**
- Modify: `schemas/org.gnome.shell.extensions.dash-to-panel.gschema.xml` (after the `context-menu-entries` key, which ends around line 770)

**Interfaces:**
- Produces: GSettings key `hide-from-panel-apps`, type `s`, default `'[]'`. Every later task reads or writes it.

- [ ] **Step 1: Create the branch**

```bash
cd ~/Dev/webdevbar/dash-to-panel
git fetch upstream master
git checkout -b feat/hide-apps-from-panel upstream/master
```

- [ ] **Step 2: Add the key**

Insert immediately after the closing `</key>` of `context-menu-entries`:

```xml
    <key type="s" name="hide-from-panel-apps">
      <default>'[]'</default>
      <summary>Apps whose windows never appear in the panel</summary>
      <description>JSON array of desktop file ids. Windows belonging to these apps are not shown in the taskbar. A pinned favourite for such an app still appears and still launches.</description>
    </key>
```

Save this as `/tmp/add-key.py` and run `python3 /tmp/add-key.py`. Write it with a
file-writing tool, not a shell heredoc — the script contains its own quoting and a
nested heredoc would terminate early.

```python
import io

p = 'schemas/org.gnome.shell.extensions.dash-to-panel.gschema.xml'
s = io.open(p, encoding='utf-8').read()

anchor = '      <summary>User defined context menu entries</summary>\n    </key>\n'
key = '''    <key type="s" name="hide-from-panel-apps">
      <default>'[]'</default>
      <summary>Apps whose windows never appear in the panel</summary>
      <description>JSON array of desktop file ids. Windows belonging to these apps are not shown in the taskbar. A pinned favourite for such an app still appears and still launches.</description>
    </key>
'''

assert s.count(anchor) == 1, s.count(anchor)
io.open(p, 'w', encoding='utf-8').write(s.replace(anchor, anchor + key))
print('key added')
```

- [ ] **Step 3: Verify the schema compiles**

```bash
glib-compile-schemas --strict --dry-run schemas/
```

Expected: no output, exit 0. A syntax error here prints the offending line and exits non-zero.

- [ ] **Step 4: Verify the key is readable**

```bash
glib-compile-schemas schemas/
gsettings --schemadir schemas/ get org.gnome.shell.extensions.dash-to-panel hide-from-panel-apps
```

Expected: `'[]'`

- [ ] **Step 5: Commit**

```bash
git add schemas/org.gnome.shell.extensions.dash-to-panel.gschema.xml
git commit -m "feat: add hide-from-panel-apps setting"
```

---

### Task 2: The exclusion filter

**Files:**
- Modify: `src/appIcons.js` — add the helper above `getInterestingWindows` (currently line 1854), and the filter inside it

**Interfaces:**
- Consumes: `hide-from-panel-apps` from Task 1. `SETTINGS` and `tracker` are already imported at `src/appIcons.js:44-53`; do not add imports.
- Produces: `export function isHiddenFromPanel(app)` — takes a `Shell.App` or `null`, returns `boolean`. Task 3 does not call it; nothing else needs it.

- [ ] **Step 1: Add the helper**

Insert immediately above `export function getInterestingWindows`:

```js
// The parsed set is cached against the RAW setting string rather than
// invalidated by a signal. A module-level cache outlives the extension - GNOME
// keeps imported modules for the life of the shell process, while
// Taskbar.destroy() disconnects the setting handlers - so a list edited while
// the extension is disabled would otherwise be read stale forever.
let hiddenAppsRaw = null
let hiddenApps = new Set()

export function isHiddenFromPanel(app) {
  let raw = SETTINGS.get_string('hide-from-panel-apps')

  if (raw !== hiddenAppsRaw) {
    hiddenAppsRaw = raw

    try {
      hiddenApps = new Set(JSON.parse(raw))
    } catch {
      // A hand-edited value must not take the panel down with it.
      hiddenApps = new Set()
    }
  }

  // An unresolved window has no stable identity, so it can never have been
  // added to the list. Keeping it is the safe direction: filtering it would
  // hide a window the user never excluded.
  return !!app && hiddenApps.has(app.get_id())
}
```

- [ ] **Step 2: Hook it into the filter**

Replace the opening of `getInterestingWindows` so it reads:

```js
export function getInterestingWindows(app, monitor, isolateMonitors) {
  let hidden = app
    ? isHiddenFromPanel(app)
    : null

  let windows = (app ? app.get_windows() : Utils.getAllMetaWindows()).filter(
    (w) =>
      !w.skip_taskbar &&
      // `app` is null on the split-app path (taskbar.js), where each window
      // must be resolved individually. get_window_app() can itself return
      // null - guarded inside isHiddenFromPanel.
      !(hidden === null ? isHiddenFromPanel(tracker.get_window_app(w)) : hidden),
  )
```

- [ ] **Step 3: Lint**

```bash
npm run lint
```

Expected: exits 0, and `git diff` shows only your change plus any prettier reformatting of it.

- [ ] **Step 4: Verify the helper's logic without GNOME**

**This is a logic rehearsal, NOT a test of the code you just wrote.** The repo has
no test runner, and `isHiddenFromPanel` cannot be imported outside GNOME because
`src/appIcons.js` imports `gi://` modules. The script below re-states the caching
logic to prove the ALGORITHM is right; it cannot catch a transcription error in
the file. Step 4b covers syntax, and Task 5 is what actually exercises the shipped
code — do not treat a green run here as verification:

```bash
cat > /tmp/hidden-check.js <<'EOF'
let hiddenAppsRaw = null
let hiddenApps = new Set()
const get = (raw, id) => {
  if (raw !== hiddenAppsRaw) {
    hiddenAppsRaw = raw
    try { hiddenApps = new Set(JSON.parse(raw)) } catch { hiddenApps = new Set() }
  }
  return !!id && hiddenApps.has(id)
}
console.log('empty  ->', get('[]', 'a.desktop') === false)
console.log('listed ->', get('["a.desktop"]', 'a.desktop') === true)
console.log('other  ->', get('["a.desktop"]', 'b.desktop') === false)
console.log('null   ->', get('["a.desktop"]', null) === false)
console.log('broken ->', get('not json', 'a.desktop') === false)
console.log('reparse->', get('["b.desktop"]', 'b.desktop') === true)
EOF
node /tmp/hidden-check.js
```

Expected: six lines, all `true`.

- [ ] **Step 4b: Syntax-check the file you actually edited**

```bash
node --check src/appIcons.js
```

Expected: no output, exit 0. This is what catches a mistyped ternary or an unbalanced
paren in Step 2. It does not run the code — nothing outside GNOME can.

- [ ] **Step 5: Commit**

```bash
git add src/appIcons.js
git commit -m "feat: filter hidden apps out of the taskbar's window list"
```

---

### Task 3: Redisplay on change

**Files:**
- Modify: `src/taskbar.js` — the `changed::` handler block that begins around line 385

**Interfaces:**
- Consumes: the setting from Task 1. Does not call Task 2's helper — the filter re-reads the setting itself.

- [ ] **Step 1: Add the handler**

In the `_signalsHandler.add(...)` list, alongside the existing entries, add:

```js
      [
        SETTINGS,
        'changed::hide-from-panel-apps',
        () => this._redisplay(),
      ],
```

Place it immediately after the `'changed::group-apps'` entry so related handlers stay together.

- [ ] **Step 2: Confirm `_redisplay` is the right method here**

```bash
grep -n "_redisplay()\|_queueRedisplay()" src/taskbar.js | head
```

Expected: both exist. The neighbouring `changed::show-favorites` handler calls `this._redisplay()` directly — match that, do not invent a different call.

- [ ] **Step 3: Lint**

```bash
npm run lint
```

Expected: exits 0.

- [ ] **Step 4: Commit**

```bash
git add src/taskbar.js
git commit -m "feat: redisplay the taskbar when the hidden-apps list changes"
```

---

### Task 4: Preferences UI

**Files:**
- Modify: `ui/SettingsFineTune.ui` — a new group before the closing `</object>` of the last `AdwPreferencesPage`
- Modify: `src/prefs.js` — inside `_bindSettings()`, **after line 2993** where `createButton` is defined

**Interfaces:**
- Consumes: the setting from Task 1, and `createButton(icon, tooltip_text, clicked)` — a `let` local to `_bindSettings()` at line 2993. Code placed before that line cannot see it.
- Produces: nothing other tasks consume.

- [ ] **Step 1: Add the group to the .ui file**

Insert as the last `<child>` of the final `AdwPreferencesGroup`'s parent page, mirroring `ui/SettingsAction.ui` lines 177-198:

```xml
    <child>
      <object class="AdwPreferencesGroup" id="hide_apps_group">
        <property name="title" translatable="yes">Hide apps from the panel</property>
        <property name="description" translatable="yes">Windows of these apps never appear in the taskbar. A pinned favourite still appears and still launches.</property>
        <child>
          <object class="GtkButton" id="hide_apps_add_button">
            <property name="halign">center</property>
            <property name="margin-top">10</property>
            <property name="receives-default">True</property>
            <property name="valign">center</property>
            <property name="width-request">100</property>
            <child>
              <object class="GtkImage">
                <property name="icon-name">list-add-symbolic</property>
                <property name="tooltip-text" translatable="yes">Add app</property>
              </object>
            </child>
            <style>
              <class name="circular"/>
            </style>
          </object>
        </child>
      </object>
    </child>
```

- [ ] **Step 2: Verify the .ui file still parses**

`gtk4-builder-tool` is NOT installed on this machine - it ships in `gtk4-devel`.
Either install it, which is the stronger check because it resolves widget classes
and properties:

```bash
sudo dnf install -y gtk4-devel
gtk4-builder-tool validate ui/SettingsFineTune.ui
```

or fall back to well-formedness only, which needs no sudo but will NOT catch a
misspelled property or a class that does not exist:

```bash
xmllint --noout ui/SettingsFineTune.ui
```

Expected either way: no output, exit 0. If you take the fallback, note that Step 5's
lint does not cover `.ui` files either, so the first real check of this markup is
opening Preferences in Task 5.

- [ ] **Step 3: Add the prefs logic**

Insert in `_bindSettings()` after the context-menu block (which ends around line 3090, comfortably past `createButton` at 2993):

```js
    // Every callback below is an arrow function, so `this` stays the prefs
    // object and `this._settings` resolves. prefs.js has no SETTINGS global.
    let hideAppsRows = []
    let hideAppsGroup = this._builder.get_object('hide_apps_group')
    let hideAppsAddButton = this._builder.get_object('hide_apps_add_button')
    let hiddenApps = []

    try {
      hiddenApps = JSON.parse(
        this._settings.get_string('hide-from-panel-apps'),
      )
    } catch {
      hiddenApps = []
    }

    let saveHiddenApps = () => {
      this._settings.set_string(
        'hide-from-panel-apps',
        JSON.stringify(hiddenApps),
      )
      createHiddenAppRows()
    }

    // A stored id whose app has been uninstalled still gets a row, showing the
    // raw id and a fallback icon, so it can be seen and removed rather than
    // silently occupying the list.
    let createHiddenAppRows = () => {
      hideAppsRows.forEach((r) => hideAppsGroup.remove(r))
      hideAppsRows = []

      hiddenApps.forEach((id, i) => {
        // GioUnix, not Gio - Gio.DesktopAppInfo is deprecated and prefs.js
        // already imports GioUnix.
        let info = GioUnix.DesktopAppInfo.new(id)
        let row = new Adw.ActionRow({
          title: info ? info.get_display_name() : id,
          subtitle: info ? id : _('Not installed'),
        })

        row.add_prefix(
          new Gtk.Image({
            gicon: info ? info.get_icon() : null,
            icon_name: info ? null : 'application-x-executable-symbolic',
            pixel_size: 32,
          }),
        )

        row.add_suffix(
          createButton('user-trash-symbolic', _('Remove'), () => {
            hiddenApps.splice(i, 1)
            saveHiddenApps()
          }),
        )

        hideAppsGroup.add(row)
        hideAppsRows.push(row)
      })
    }

    hideAppsAddButton.connect('clicked', () => {
      // Only apps the user could see in the app grid, minus those already
      // listed, so the same app cannot be added twice.
      let apps = Gio.AppInfo.get_all()
        .filter((a) => a.should_show() && hiddenApps.indexOf(a.get_id()) < 0)
        .sort((a, b) => a.get_display_name().localeCompare(b.get_display_name()))

      if (!apps.length) return

      let model = new Gio.ListStore({ item_type: Gio.AppInfo })
      apps.forEach((a) => model.append(a))

      // One factory renders BOTH the selected value and the popup rows.
      // list-factory is deliberately left unset so the two cannot drift.
      let factory = new Gtk.SignalListItemFactory()

      factory.connect('setup', (f, item) => {
        let box = new Gtk.Box({ spacing: 8 })

        box.append(new Gtk.Image({ pixel_size: 16 }))
        box.append(new Gtk.Label({ xalign: 0 }))
        item.set_child(box)
      })

      factory.connect('bind', (f, item) => {
        let app = item.get_item()
        let box = item.get_child()

        box.get_first_child().set_from_gicon(app.get_icon())
        box.get_last_child().set_label(app.get_display_name())
      })

      let combo = new Adw.ComboRow({
        title: _('Add app'),
        model,
        factory,
        // enable-search alone has no search key when the model holds objects
        // rather than strings; the expression is what supplies it.
        expression: Gtk.ClosureExpression.new(
          GObject.TYPE_STRING,
          (app) => app.get_display_name(),
          null,
        ),
        enable_search: true,
      })

      const ADD_RESPONSE = 'add'
      let dialog = new Adw.AlertDialog({
        heading: _('Hide an app from the panel'),
      })
      let list = new Gtk.ListBox({ selection_mode: Gtk.SelectionMode.NONE })

      list.add_css_class('boxed-list')
      list.append(combo)
      dialog.set_extra_child(list)
      dialog.add_response('cancel', _('Cancel'))
      dialog.add_response(ADD_RESPONSE, _('Add'))
      dialog.set_response_appearance(
        ADD_RESPONSE,
        Adw.ResponseAppearance.SUGGESTED,
      )

      dialog.connect('response', (d, response) => {
        if (response != ADD_RESPONSE) return

        hiddenApps.push(model.get_item(combo.get_selected()).get_id())
        saveHiddenApps()
      })

      dialog.present(hideAppsAddButton.get_root())
    })

    createHiddenAppRows()
```

- [ ] **Step 4: Confirm every import used above already exists in prefs.js**

```bash
grep -n "^import" src/prefs.js | head -12
```

`Adw`, `Gtk`, `Gio`, `GioUnix` and `GObject` must all be present (lines 23-30). If any is missing, add it in the same style as its neighbours **before** proceeding — the code above will throw a ReferenceError otherwise.

- [ ] **Step 5: Lint**

```bash
npm run lint
```

Expected: exits 0.

- [ ] **Step 6: Commit**

```bash
git add ui/SettingsFineTune.ui src/prefs.js
git commit -m "feat: Fine-Tune list for apps hidden from the panel"
```

---

### Task 5: Manual verification

**Files:** none — this task changes nothing. It is the gate before the PR.

**Interfaces:**
- Consumes: Tasks 1-4, all committed.

Install and relog in first. Editing `src/*.js` and re-enabling the extension is **not** enough; GNOME caches the modules.

```bash
make install     # or: cp -r src schemas ui metadata.json ~/.local/share/gnome-shell/extensions/dash-to-panel@jderose9.github.com/
```

Then log out and back in.

**Prerequisite.** The steps below use Smile as the app to hide. It is not part of this
repository. Either install it:

```bash
flatpak remote-add --user --if-not-exists flathub https://dl.flathub.org/repo/flathub.flatpakrepo
flatpak install --user -y flathub it.mijorus.smile
```

and bind `Super+.` to `flatpak run it.mijorus.smile` in Settings → Keyboard → Custom
Shortcuts, or substitute **any** installed app you can open and close freely — a text
editor works. Only steps 1 and 5 care that it opens quickly; nothing depends on Smile
specifically.

- [ ] **Step 1: Basic exclusion**

Open Preferences → Fine-Tune → Hide apps from the panel → `+`. Confirm the picker shows an **icon beside every app name**, that typing filters the list, and add Smile.
Open Smile with `Super+.`.
Expected: Smile's window does not appear on either panel. Auto-paste still works.

- [ ] **Step 2: Pinned favourite survives**

Pin Smile to the panel, then open it.
Expected: the pinned launcher is visible and launches; opening it adds no running indicator and no second icon.

- [ ] **Step 3: Live removal**

With Smile still open, remove it from the list.
Expected: its icon reappears immediately, without touching another window and without a relogin. This is what proves the redisplay is queued rather than only the cache cleared.

- [ ] **Step 4: No duplicates in the picker**

Re-add Smile, then open the picker again.
Expected: Smile is absent from the dropdown.

- [ ] **Step 5: Split-app path**

Set `group-apps` **off** and `group-apps-use-launchers` **on** (or `show-favorites` off). This is the only path calling `getInterestingWindows(null, ...)`.
Expected: Smile is still hidden.

- [ ] **Step 6: Stale-cache path**

Disable Dash to Panel, change the list, re-enable it.
Expected: the new list applies immediately. This is the case the raw-value cache key exists for.

- [ ] **Step 7: Uninstalled app**

```bash
gsettings --schemadir schemas/ set org.gnome.shell.extensions.dash-to-panel hide-from-panel-apps '["does.not.exist.desktop"]'
```

Reopen Preferences.
Expected: a row appears showing the raw id, a fallback icon, and "Not installed"; removing it does not throw. Check `journalctl --user -b -o cat | grep -i "JS ERROR"` is clean afterwards.

- [ ] **Step 8: Restore your own settings and open the PR**

Write the PR body first — `gh pr create` fails if the file does not exist:

```bash
cat > /tmp/pr-body.md <<'EOF'
Some windows have no business occupying a taskbar slot - an emoji picker summoned
with a shortcut, used for two seconds, then dismissed. On Wayland the application
cannot say so itself: `MetaWindow.skip-taskbar` is read-only (`flags: 225,
writable: False` on Mutter 18) and xdg-shell carries no client request for it. The
panel is the only place left to solve it.

This adds a `hide-from-panel-apps` setting - a JSON array of desktop file ids - and
a list in the Fine-Tune tab to manage it, with a searchable app picker showing each
app's icon.

A pinned favourite for an excluded app deliberately still appears and still
launches; only the running-window icon is suppressed. Pinning is a statement about
the launcher, excluding is a statement about running windows, and filtering the pin
away too would look like the pin had failed.

Alt-Tab is out of scope - it reads the same read-only property and is a different
component.

Notes:
- `getInterestingWindows()` is called with `app === null` on the split-app path, so
  windows are resolved individually via `get_window_app()`, which can itself return
  null. Unresolved windows are kept, never filtered.
- The parsed set is cached against the raw setting string rather than invalidated by
  a signal, because a module-level cache outlives the extension across
  disable/enable while `Taskbar.destroy()` disconnects the handlers.

Tested manually on GNOME Shell <VERSION> / Wayland with panels on two monitors:
basic exclusion, pinned favourite unaffected, live removal, no duplicates in the
picker, the split-app path, disable-edit-re-enable, and an uninstalled desktop id.
There is no automated test suite in this repository.
EOF
```

Replace `<VERSION>` with the output of `gnome-shell --version`. Then:

```bash
gsettings --schemadir schemas/ set org.gnome.shell.extensions.dash-to-panel hide-from-panel-apps '[]'
git push -u origin feat/hide-apps-from-panel
gh pr create --repo home-sweet-gnome/dash-to-panel --base master \
  --head WebDevBar:feat/hide-apps-from-panel \
  --title "feat: hide chosen apps from the panel" --body-file /tmp/pr-body.md
```


- [ ] **Step 9: Merge into the integration branch**

```bash
git checkout webdevbar-local
git merge --no-edit feat/hide-apps-from-panel
git push
```
