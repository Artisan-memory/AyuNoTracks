# AyuGram Desktop — no-trackers build

A personal fork of [AyuGram Desktop](https://github.com/AyuGram/AyuGramDesktop) with
every network path back to servers run by the fork author severed.

Upstream is tracked closely — this fork carries a small set of surgical patches on
top of the current AyuGram `dev` branch.

## What is removed

- **Autoupdate.** The client used to check `update.ayugram.one` periodically and
  fetch signed binary updates from there. Disabled at compile time via
  `DESKTOP_APP_DISABLE_AUTOUPDATE`. To update, download a new build from this
  repo's releases page and replace the exe.
- **Crash dumps.** The crash reporter previously uploaded minidumps to
  `sentry.radolyn.com` when the user clicked "Send crash report". A minidump is a
  snapshot of process memory and can contain message text. Breakpad is no longer
  linked (`DESKTOP_APP_DISABLE_CRASH_REPORTS`), and the upload code path is
  additionally patched out so nothing leaves the machine even if the crash window
  opens.
- **The remote-config beacon.** `RCManager` used to `GET update.ayugram.one/rc`
  every hour from app start (with `api.exteragram.app` as fallback). The response
  only carried developer/supporter ID lists, badge data and donation amounts, so
  dropping the request costs nothing: the built-in fallback lists take over.
  Effect: no periodic contact with any AyuGram-controlled server, and no leaked
  IP or usage signal on startup.

## What is added or changed

- **Per-account notification silence.** A new setting under _AyuGram Preferences
  → General_ that, for the currently signed-in account, can suppress
  notifications entirely or only for groups and channels. Enforced at the
  earliest possible point in the notification dispatch pipeline
  (`Window::Notifications::System::schedule`), so it applies uniformly to sound,
  tray, taskbar and OS toasts. The setting is stored per-account: switch
  account, set independently.
- **Local developer/supporter badge.** The badge visible next to your own name
  in your own client. Client-side only, cosmetic, invisible to anyone else —
  same mechanism AyuGram uses for its own badge ID list, just extended to include
  the signed-in account.
- **"Reset settings" and "Register URL scheme" live under General.** The whole
  _Other_ section is gone from the sidebar; those two useful actions moved to
  the bottom of _General_.
- **Link section repointed.** In _AyuGram Preferences → Links_ the "Channel"
  entry opens `@ayunotrack` and the "Source on GitHub" entry opens this
  repository. Chat and Crowdin left alone.

## What is _not_ touched

Everything AyuGram is actually good for is preserved as-is:

- Ghost mode and its per-account granularity (read receipts, typing, online,
  upload progress, offline-after-online).
- Local database of deleted and edited messages (`tdata/ayudata.db`). Only
  reactively populated — an incoming message hits this database only when a
  delete or edit event actually arrives for it, and only if the corresponding
  toggle is on.
- The `is_noforwards()` / `is_restricted()` bypass that lets you save and
  forward from channels that opted out.
- Streamer mode, filters, translator, message history UI, everything else.

## What is _not_ removed and worth knowing

- **Language pack download.** On startup the client fetches
  `cdn.jsdelivr.net/gh/AyuGram/Languages@l10n_main/values/langs/<lang>/Shared.json`
  once and caches it to `tdata/ayu/languages/`. The file only ever writes keys
  under the `ayu_` prefix — it cannot overwrite Telegram's own strings. Not on
  an AyuGram server, but does leak your IP and language to jsDelivr/Fastly one
  time per launch. Left in because the feature is useful and the exposure is
  minor.
- **Regdate lookup.** Opening a foreign profile still sends the target user id
  to `@ayugrambot` or `@exteraAuthBot` (inline query over MTProto) to get the
  registration date. Not changed here; those bots receive whose profile you
  looked up and when.
- **Translation.** Sends message text to Google or Yandex if you invoke it.
  That's inherent to the feature.

## Building

Follow [docs/building-win.md](docs/building-win.md) upstream. The only extra
flags this build uses at configure time:

```
-D DESKTOP_APP_DISABLE_AUTOUPDATE=ON
-D DESKTOP_APP_DISABLE_CRASH_REPORTS=ON
```

The exe is fully static (Qt, OpenSSL, FFmpeg, everything). No installer. Drop
it in a folder inside your user profile, keep `tdata\` next to it — it is
portable that way.

## The diff, in one glance

```
Telegram/Resources/langs/lang.strings              (+4)   new UI strings
Telegram/SourceFiles/ayu/ayu_infra.cpp             (-1)   RCManager beacon killed
Telegram/SourceFiles/ayu/ayu_settings.{h,cpp}      (+~30) per-account mute setting
Telegram/SourceFiles/ayu/ui/settings/settings_general.cpp
                                                   (+~60) mute UI + reset/register-url moved here
Telegram/SourceFiles/ayu/ui/settings/settings_main.cpp
                                                   (~4)   drop Other category; repoint channel + docs links
Telegram/SourceFiles/ayu/utils/telegram_helpers.cpp
                                                   (+15)  local self badge
Telegram/SourceFiles/core/crash_report_window.cpp  (-3)   sentry upload path disarmed
Telegram/SourceFiles/window/notifications_manager.cpp
                                                   (+17)  per-account mute gate at schedule()
```

Attribution: everything AyuGram is left in place — the LEGAL and LICENSE files
stay unchanged, and this fork exists as a fork so that lineage is visible.
