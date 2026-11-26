# CleanFrame on my Samsung QE50Q65A

A copy of [CleanFrame](https://github.com/nthnclrk/CleanFrame) 0.7.0 (commit 845f332), with my own changes. MIT licence; see `LICENSE` and `THIRD_PARTY_NOTICES.md`.

## The TV

| | |
|---|---|
| Model | QE50Q65AAUXXU |
| Firmware | T-NKLADEUC-2310.0 |
| Tizen | 6.0, armv7 |
| TV IP address | 192.168.1.122 (port 26101) |
| App ID | `Vjn9LLyyrJ.CleanFrame` (package `Vjn9LLyyrJ`) |
| Certificate profile | `CleanFrame` (files in `C:\Users\<you>\SamsungCertificate\CleanFrame`) |

Native Home replacement is not possible on this TV: `intershell_support` is disabled. Open CleanFrame from Apps.

## My changes (version 0.7.2)

- 24-hour clock with a leading zero (`app/main.js`, `clockText`)
- Art Mode button removed from the top bar (`app/main.js`, `utils`)
- White marks for BBC iPlayer, ITVX and Channel 4 (`app/assets/icons/`, `app/main.js` `PRIMARY_APPS`, `app/device.js` `IDS`, `app/assets/icons/metrics.js`)
  - Channel 4 and ITVX come from Simple Icons 16.38.0 (CC0)
  - BBC iPlayer is the play symbol traced from the app icon
- TizenTube (sideloaded, `xvvl3S1TT1.TizenTubeStandalone`) uses the YouTube mark with a small T badge (`app/assets/icons/tizentube.svg`)
- Saved Home entries stored by TV app ID move to the new branded apps (`app/main.js`, `clean`)
- Version raised to 0.7.1 (`app/config.xml`)

## Building

The project's own build and install scripts do not run on Windows. Build the unsigned widget on Mac, Linux or WSL with Python 3.10+:

```
cd cleanframe
mkdir -p dist
python3 -c "import sys; sys.path.insert(0,'scripts'); from font_assets import ROOT, write_widget_archive; write_widget_archive(ROOT/'dist/CleanFrame-unsigned.wgt', include_private=False, identity={'package_id':'Vjn9LLyyrJ','app_id':'Vjn9LLyyrJ.CleanFrame'})"
```

After changing an icon, rebuild `app/assets/icons/metrics.js` with `node scripts/build-icon-metrics.mjs` (needs `npm ci`), then copy `.local/materials/metrics.js` over it.

## Signing and installing on Windows

Copy `dist/CleanFrame-unsigned.wgt` to `C:\CleanFrame`, empty `C:\CleanFrame\Signed`, then run each line in Command Prompt:

```
cd C:\tizen-studio\tools\ide\bin
tizen package -t wgt -s CleanFrame -o C:\CleanFrame\Signed -- C:\CleanFrame\CleanFrame-unsigned.wgt
cd C:\tizen-studio\tools
sdb connect 192.168.1.122:26101
sdb -s 192.168.1.122:26101 push C:\CleanFrame\Signed\CleanFrame-unsigned.wgt /home/owner/share/tmp/sdk_tools/CleanFrame.wgt
sdb -s 192.168.1.122:26101 shell 0 vd_appinstall Vjn9LLyyrJ.CleanFrame /home/owner/share/tmp/sdk_tools/CleanFrame.wgt
sdb -s 192.168.1.122:26101 shell 0 execute Vjn9LLyyrJ.CleanFrame
```

Expect `is created successfully`, `1 file(s) pushed`, `install completed` and `successfully launched`. Plain `sdb install` does not install on this TV.

Keep the same app ID and certificate for every update, or the TV installs a second copy.
