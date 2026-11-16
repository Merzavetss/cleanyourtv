# KIVI 65U710KB (`bangbae`, MediaTek m7322, Android TV 11): the Netflix key crashes the TV service and the picture settings stop applying

Owner report, one unit, no issue number. A debloated set on the vendor MediaTek stack, launcher
installed and Home, Netflix disabled by choice. Pressing the remote's Netflix key afterwards took
the TV's settings service down, and nothing said so: the picture and sound menus kept saving their
numbers while the panel did not change, until the owner rebooted the next day.

## Identify the firmware

| Field | Reported value |
|---|---|
| Retail model | KIVI 65U710KB, sold as "KIVI 4K Android TV" |
| Product / device | `MTCMSD6886EU_KIVI` / `bangbae` |
| SoC | MediaTek `m7322` (`ro.board.platform`, `ro.hardware`) |
| Android / SDK | Android TV 11 / 30 - not Google TV |
| `ro.build.display.id` | `RTO8.260427.001` |
| Security patch | 2025-11-01 |
| RAM | 1,349 MB |
| Stock Home | `com.google.android.tvlauncher` (disabled once the launcher was Home) |
| Vendor stack | `com.mediatek.tv.service`, `com.mediatek.tvinput`, `com.mstar.*` apps |

It is an MStar-named MediaTek stack and `com.cvte.*` is absent, so the CVTE notes in `examples/`
(including the rule that a CVTE board keeps its own launcher enabled) do not apply here.

## The symptom

Contrast, brightness, saturation and backlight sliders save: `settings get global picture_contrast`
answered 90 after the owner had moved the slider, and the image did not change. The owner read it as
the debloat having broken the picture settings, which is close: the settings UI was working, the
half that applies the value was not.

## Where the value goes

The settings app is honest about it in its own log. `PictureFragment.onPreferenceChange` ends in
`putSettingValueInt`, and nothing else follows - no `TV_MtkTvConfig` or `setConfigValue` line:

```
21:10:46.922 E PictureFragment: onPreferenceChange preference == picture_contrast  73
21:10:46.929 E PictureFragment: isNotFirstChange = 1
```

The other half is `com.mediatek.tv.service/com.mediatek.tv.agent.TVMenuSettingsService`
(`TvServices.apk`): its `mSettingObserver` watches the same `Settings.Global` keys and pushes each
one to the TV API. `picture_auto_backlight` and `picture_mode` do not depend on it - the fragment
calls `TVSettingConfig` for those two itself - while the other four keys have no other writer. The
service was not running:

```
* ServiceRecord{ccf3f5d ... TVMenuSettingsService}
    app=null   startRequested=false   crashCount=1
    lastActivity=-21h33m
```

and `system_server` had already given up on restarting it:

```
2026-11-15 17:33:29 system_server_wtf
android.util.Log$TerribleFailure: Restarting service that is not needed:
  ServiceRecord{ccf3f5d u0 com.mediatek.tv.service/com.mediatek.tv.agent.TVMenuSettingsService}
    at com.android.server.am.ActiveServices.performServiceRestartLocked
```

## Why it died

The dropbox has the crash from the same second, and the trigger was the remote:

```
2026-11-15 17:33:29 system_app_crash
Process: com.mediatek.tv.service (PID 3548, UID 1000)
java.lang.RuntimeException: Unable to start receiver com.mstar.netflixobserver.NetflixKeyReceiver:
  android.content.ActivityNotFoundException: No Activity found to handle
  Intent { act=com.netflix.action.NETFLIX_KEY_START flg=0x10020000 pkg=com.netflix.ninja }
    at com.mstar.netflixobserver.NetflixObserver.resumeNetflix(NetflixObserver.java:285)
    at com.mstar.netflixobserver.NetflixKeyReceiver.onReceive(NetflixKeyReceiver.java:128)
```

The Netflix button arrives as `KEYCODE_MTKIR_NETFLIX` (265) from
`/system/usr/keylayout/ttxkeymap.ini`, the vendor layer turns it into an
`android.intent.action.GLOBAL_BUTTON` broadcast, and `NetflixKeyReceiver` - a receiver inside
`com.mediatek.tv.service`, running as uid 1000 - starts `com.netflix.action.NETFLIX_KEY_START` on
`com.netflix.ninja`. The debloat had disabled that package, `resumeNetflix` calls `startActivity`
without a `try`/`catch`, and the exception leaves `onReceive`: the framework counts that as a
crashed receiver and takes the whole `com.mediatek.tv.service` process down with it. It is restarted
for Live TV and HDMI, but not `TVMenuSettingsService` - `performServiceRestartLocked` treats the
extra services as not needed.

## Recovery

A reboot. The service is started at boot (its `createTime` was the boot time; it had been up three
days when it died) and reads the stored values, so the numbers the owner had already saved came back
as they were. Confirm it is alive before chasing a picture problem any further:

```sh
tools/tv.py 'dumpsys activity services com.mediatek.tv.service | grep -A 6 TVMenuSettingsService'
```

`app=ProcessRecord{... com.mediatek.tv.service/1000}` is what a working one looks like; `app=null`
is the broken state. The same symptom and recovery are in
[docs/TROUBLESHOOTING.md](../TROUBLESHOOTING.md#the-netflix-button-kills-the-picture-settings-until-a-reboot-kivi-mediatek).

## Disarming the key: what does not work

None of these worked on this build, and each one is cheap to rule out again:

- **Disabling the receiver.** `pm disable-user --user 0
  com.mediatek.tv.service/com.mstar.netflixobserver.NetflixKeyReceiver` is refused:
  ```text
  java.lang.SecurityException: Shell cannot change component state for
    com.mediatek.tv.service/com.mstar.netflixobserver.NetflixKeyReceiver to 3
  ```
  The shell cannot change the enabled state of a component of a system package, and `adb root`
  answers `adbd cannot run as root in production builds`. Disabling the whole package is not an
  option - it is the tuner, the inputs and Live TV.
- **Remapping the button.** The vendor hotkey dispatcher (`com.mediatek.hotkey.dispatcher`) reads
  `/vendor/etc/hot-key-info.ini`, which maps only Amazon, Google Play and the IR assist key, so there
  is no Netflix entry to change; `/vendor` is mounted `ro` (`/dev/block/dm-2 ... ext4 (ro,...)`), and
  `/data/system/devices/keylayout`, the writable override, does not exist on this firmware.
- **Making `resumeNetflix` return early.** Its OAD branch reads `vendor.mtk.3rd.flag` through
  `SaveValue.readWorldStringValue` from `content://com.mediatek.tv.internal.data/global_value`, but the
  same string is referenced by `MtcLiveTVSetupWizard`, `LiveTV`, `TvSettingsPlus` and
  `MtcFactoryCommandService`: it means "firmware update in progress" for the whole TV, not a per-app
  switch. Not worth it.

## The two ways out

1. Leave the key armed and do not press it; if it is pressed, reboot, which keeps the values.
2. `pm enable --user 0 com.netflix.ninja`: `com.netflix.ninja/.MainActivity` declares
   `com.netflix.action.NETFLIX_KEY_START`, so the intent resolves and the receiver cannot throw. The
   key then opens Netflix, and `pm disable-user --user 0 com.netflix.ninja` puts it back. Choose this
   only if the owner is prepared to see Netflix start - a disabled streaming app does not disarm its
   remote button.

## For the same board

Where a list in `examples/` handles this button, it does so by disabling the package
`com.mstar.netflixobserver` (`examples/xiaomi-mitv-mssp3`, `examples/hisense-hg9`) or by leaving
that line commented out (`examples/panasonic-4ktv-0su`). On this build no package carries the name -
it is only a class prefix inside `com.mediatek.tv.service` - so a reader comparing package lists
finds nothing to switch off and the key stays armed. Check which of the two a set has before
applying a list:

```sh
tools/tv.py 'pm list packages | grep netflixobserver'
tools/tv.py 'dumpsys package com.mediatek.tv.service | grep -i NetflixKeyReceiver'
```

A package answer means the list can decide; only the second line means the key cannot be disabled
from the shell, and the owner has to live with it or keep the app enabled.

The rest of the cleanup on this unit - fifty packages, launcher as Home, reboot-checked - is not
published as a list. The vendor stack it shares (`com.mediatek.tvinput`, `com.mediatek.tv.agent`,
MediaTek, stock `com.google.android.tvlauncher`) is closest to
[`examples/oneplus-tv-cebu/`](../../examples/oneplus-tv-cebu/debloat.txt), which is tested on
another model and should be applied here only name by name.
