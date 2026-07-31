# WebDevBar fork notes - dash-to-panel

Why this fork exists, what it carries, and what is planned. Not upstream documentation.
Started 2026-07-30 on Fedora 44 / GNOME Shell 50.0 (Wayland).

## Branches - build from `webdevbar-local`, NOT from a feature branch

| Branch | Purpose |
|---|---|
| **`webdevbar-local`** | **what we RUN.** master + every feature/fix merged. Always `make install-local` from here |
| `master` | upstream master + #2493 + our hoist + #2509 |
| `feat/status-group-padding` | PR #2531, single-purpose |
| `fix/disposal-orphaned-clone` | orphaned-clone + hover-timeout fix, not yet submitted |
| `fix/badge-double-count` | badge took the SUM of the Unity and MessageTray counts, now the max. Not yet submitted |
| `test/signal-disposal-2469` | upstream PR #2469 applied for evaluation. Reviews say it does NOT fix our paths |

⚠ **Building from a single feature branch silently drops every other patch.** On 2026-07-30 a build
from `fix/disposal-orphaned-clone` (cut from master) removed the Status Group Padding feature
entirely - the slider vanished from prefs after a relogin. The dconf value survived, so nothing was
lost, but it looked like the feature had failed to persist. Keep PR branches single-purpose and
merge them into `webdevbar-local` for daily use.

**Refresh it whenever a branch changes, then install from it:**

```bash
git checkout webdevbar-local
git merge --no-ff --no-edit master feat/status-group-padding fix/disposal-orphaned-clone
make install-local
```

Any new branch that is ready to use gets merged here too, in the same command. Relogin after.

## What this fork carries

`master` is upstream `master` plus:

| Commits | What | Upstream status |
|---|---|---|
| `c3941a9`…`03ade4a` (6) | PR **#2493** - `startupPreparedHandler` for overview control on GNOME 50, cherry-picked with authorship preserved | **open** |
| `41c4b87` | Our own: avoid a redundant `hasOverview` set-and-revert in the already-started branch | not submitted |
| `b1db153` (merge `4b1548f`) | PR **#2509** - our `+0/-2`, badges for windowless apps | **open** since 2026-06-02 |

Build and install with `make install-local` (→ `~/.local/share/gnome-shell/extensions/`, which wins
over the Fedora RPM's `/usr/share`). **Do not install the Fedora RPM alongside** - same UUID.
GNOME caches extension JS, so a code change needs a full relogin on Wayland, not just
disable/enable.

### #2493 - verified working, with a known ceiling

Tested by relogin on GNOME 50: no overview at login, but **the overview *background* is briefly
visible and then animates away.** That is not a shortfall of this PR. Three independent
implementations behave identically - #2493, our own earlier local patch, and Simple Taskbar's
`overviewIntegration.js` - because by the time any extension's `enable()` runs, the shell's startup
animation is already going and there is no API to stop it (the PR's own comment says so). Treated as
a ceiling; masked instead by the **Overview Background** extension (EGO 5856) with blur 0, which
makes the flash read as the desktop wallpaper.

Reviewed while cherry-picking: DtP **already** restores `hasOverview` on disable
(`src/extension.js:185`), so an earlier assumption that Simple Taskbar was more careful there was
wrong.

## 🔲 PLANNED - `notification-clear-on-focus` (not started, do not implement yet)

**Observed behaviour (2026-07-30).** DtP clears a notification count as soon as the app is focused:

- `src/notificationsMonitor.js` `_mergeState()` - `if (tracker.focus_app?.id == appId)` zeroes both
  `count` (Unity/LauncherEntry) and `trayCount` (MessageTray).
- the `notify::focus-app` handler (~lines 56-66) additionally resets the whole state to default.

Consequence, as seen in practice with Mailspring and with Brave: the badge shows unread, you open
the app once, the count zeroes, and from then on **only new arrivals** can produce a badge - the
pre-existing unread total never comes back.

**This is a defensible default** ("you have seen it"), and it is *not* a bug. So the proposal is an
**opt-in setting, not a behaviour change**:

| | |
|---|---|
| Key | `notification-clear-on-focus`, boolean, **default `true`** |
| `true` | exactly today's behaviour - nothing changes for any existing user |
| `false` | do not zero on focus; the badge keeps the emitter's count and falls as messages are read |
| Touch points | a key in `schemas/org.gnome.shell.extensions.dash-to-panel.gschema.xml`, a checkbox in `prefs.js`, and two guards in `notificationsMonitor.js` (`_mergeState` + the `notify::focus-app` handler) |

**Gate only `count`, never `trayCount`** - and say why in the PR, because a reviewer will ask:

- The Unity `count` is **authoritative and self-correcting**. Measured on the session bus: Mailspring
  emits `application://Mailspring.desktop` with `count` **descending 20 → 19 → 18** as messages are
  read, `count-visible: true`, roughly once per second. So persisting it is safe - the app itself
  drives it back to zero.
- `trayCount` comes from MessageTray banners with **no decrement signal**. Persisting it would
  produce a badge that can never clear, which is a worse bug than the one being fixed.

## The badge double-count (fixed 2026-07-31) - upstream issue #2515

`_mergeState()` computed the badge as
`((count-visible || 0) && (count || 0)) + (trayCount || 0)` - the Unity LauncherEntry count PLUS
the MessageTray count.

Mailspring drives **both** channels: it publishes a LauncherEntry count over DBus, and it is
registered for GNOME notifications (`application-children`, `enable=true`, `show-banners=true`), so
each banner raises `source.count` too. One new mail therefore showed **2** while Mailspring's own
unread counter showed 1.

**Already reported upstream as
[#2515](https://github.com/home-sweet-gnome/dash-to-panel/issues/2515)** (2026-06-23, open, no
comments), found independently via Telegram and Discord, pointing at the same lines.

**The fix is precedence, not `Math.max()`.** The first attempt here took the larger of the two. The
issue author instead proposed what dash-to-dock does, and reading dash-to-dock's
`_updateNotificationsCount()` confirms it: when the LauncherEntry count is non-zero it uses **only**
that. An app publishing its own count is authoritative about how much is unread, while banners can
be stale or several for one item. It also makes the PR easy to review, since it matches the most
widely deployed dock.

The two only disagree when the app's count is lower than the banner count:

| State | old sum | max | precedence |
|---|---|---|---|
| LauncherEntry only, 5 | 5 | 5 | 5 |
| banners only, 3 | 3 | 3 | 3 |
| both, one new message | **2** | 1 | 1 |
| 20 unread + 1 banner | **21** | 20 | 20 |
| 1 unread + 3 banners | **4** | **3** | 1 |
| all read, 3 banners pending | 3 | 3 | 3 |

Apps using a single channel are unaffected, which is why this survived so long.

dash-to-dock keeps it behind `applicationCounterOverridesNotifications`, default on. Not adding a
setting here: adding the two numbers is never meaningful, so there is nothing to opt into.

## ⚠ THE TRAP THAT COST AN AFTERNOON - GNOME caches extension modules

**`grep` on the installed file proves NOTHING about what the shell is running.** GNOME imports
extension ES modules once per shell process; `disable`/`enable` does **not** reload changed code  - 
only a full shell restart (logout/login on Wayland) does.

On 2026-07-30 this produced a completely false conclusion. The badge fix was installed at 14:10, the
shell had been running since 13:31, and the windowless badge therefore still failed. A `grep -c` on
the installed `appIcons.js` returned 0 occurrences of the guard, which "confirmed" the fix was
active - so the failure looked like proof that PR #2509 does not work on GNOME 50. Two independent
code reviews then reasoned correctly about a checkout that **was not the running code**.

After a verified relogin (`ps -o lstart -C gnome-shell` postdating the install), the identical test
passed: **the windowless badge appears, and #2509 works.**

**Before concluding any patch does not work:**

```bash
ps -o lstart= -C gnome-shell          # when did the running shell start?
stat -c '%y' ~/.local/share/gnome-shell/extensions/<uuid>/<file>.js
journalctl --user -b | grep -i "GNOME Shell started" | tail -1
```

If the shell predates the file, you are testing old code. This belongs in the "verify, don't assume"
column: a file on disk is not a loaded module, and a passing `grep` is not a passing test.

## Verified facts about the badge path (measured, not inferred)

Useful when writing either PR comment.

- **The emitter's id must match the desktop-file id exactly, case included.**
  `_handleLauncherUpdate()` strips the scheme from `application://Mailspring.desktop`;
  `_updateState()` strips `.desktop`, consults `knownIdMappings` (which has exactly ONE entry, for
  Evolution), re-appends `.desktop`, and emits `` `update-${appId}` ``; icons subscribe to
  `` `update-${this.app.id}` `` (`src/appIcons.js:305-307`). No normalisation anywhere.
- **Two defects in that same function**, both worth a separate hardening PR:
  1. matching is case-sensitive, so an emitter whose casing differs from the installed `.desktop`
     name writes state under a key no icon listens to;
  2. `appId.replace('.desktop', '')` is **unanchored** - it strips the first occurrence anywhere
     rather than the suffix. Should be `/\.desktop$/`.
  Note `Shell.AppSystem.lookup_app()` is itself exact-match, so the fix is: exact lookup first, then
  a **cached** case-insensitive scan of installed ids, invalidated on `installed-changed`. Keep
  `knownIdMappings` - it solves a different problem (daemon ids).
- **`count-visible` handling is correct**, despite looking wrong. `notificationsMonitor.js:128-130`
  does `((state['count-visible'] || 0) && (state.count || 0)) + (state.trayCount || 0)`; the boolean
  collapses to `0` on the falsy paths before the addition. Do **not** raise this upstream - two
  independent reviews confirmed it is spec-compliant, and claiming it as a bug would be refutable.
- **`LauncherEntry` is not vestigial.** The `// pretty much useless` comment above the subscription
  is wrong: that subscription is the only code path populating `count`, `count-visible` and the Unity
  half of `urgent`. `_checkNotifications()` writes only `trayCount`/`trayUrgent`, from banners that
  reset on focus. For any app publishing unread state via LauncherEntry, that subscription **is** the
  badge feature.
- **The guard #2509 removes also caused stale badges** - the early return skipped
  `_maybeUpdateNumberOverlay()`, so a badge already drawn could neither update nor clear once the
  last window closed.
- **Wording**: "windowless apps" is imprecise. `src/taskbar.js` (~1028-1031) keeps a zero-window icon
  only if the app is a favourite, so the accurate description is **"pinned apps backed by a
  background service"**.

## Mailspring-specific gotchas found while testing (not DtP bugs)

- **The app must actually be running.** Quitting Mailspring takes the emitter with it, so no badge is
  correct. It needs to sit in the tray - `~/.config/autostart/Mailspring.desktop` with
  `Exec=/usr/bin/mailspring --background`, plus Mailspring's own keep-running preference.
- **`Mailspring.desktop` needs a capital M** - the emitter hardcodes
  `application://Mailspring.desktop`. The official 1.23.0 RPM ships it correctly.
- **The RPM's launcher has no `StartupWMClass`.** GNOME's `WindowTracker` matches a window by
  lowercasing `WM_CLASS` and looking for `<wmclass>.desktop`, i.e. `mailspring.desktop`, which does
  not exist on a case-sensitive filesystem. Added `StartupWMClass=Mailspring` to the user-level
  launcher. **Possibly worth reporting to Mailspring** - without it GNOME cannot reliably match its
  own window, independent of which taskbar extension is used.
- **Mailspring has no GNOME Online Accounts integration** - zero references to `goa-1.0`,
  `org.gnome.OnlineAccounts` or `GoaClient` in `app.asar`. It ships its own OAuth client and IMAP
  engine, so accounts are authorised separately from GOA. Not a missing setting.
