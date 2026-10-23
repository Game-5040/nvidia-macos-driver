# Brightness persistence for the internal panel

macOS does not remember the internal panel's brightness with this driver: `NVRMFB`'s `publishBacklight()`
takes the level NVKMS reports at startup (100%) and there is no saved value to restore. This directory adds
that, without touching the driver.

Tested on: HP OMEN 16-ap1xxx (Ryzen 9 9955HX, RTX 5070 Laptop), macOS 26.7.1 (25G241), driver 2.5.x.

## Why it needs three pieces

The same panel exposes brightness through three different interfaces, and they behave differently:

| Interface | Behaviour |
|---|---|
| `IODisplayParameters.brightness` | Written but ignored (a slider placeholder) |
| `IODisplayParameters.linear-brightness` | Actually moves the panel, but the Control Center slider stays put |
| `DisplayServicesSetBrightness` (private framework) | Panel **and** slider, the path Control Center uses |

`DisplayServices` needs a GUI session, so a system daemon cannot use it; `linear-brightness` has no such
limit and can be written before login. Hence two roles:

```
daemon  brightness-early   before login   write linear-brightness via IOKit   -> panel restored
agent   bw (resident C)    after login    write DisplayServices             -> slider synced
          - IODisplayWrangler 1 -> 4   display-only sleep wake  -> re-apply
          - wall clock jump > 2 s      full system sleep wake    -> re-apply
          - brightness jumps to 100%   panel reset               -> re-apply
          - anything else              user change               -> save
```

## Files

| File | Purpose |
|---|---|
| `bl.c`, `blp.c` | IOKit read/write of `linear-brightness` |
| `ds.c` | DisplayServices read/write (`ds` reads, `ds 0.4` writes) |
| `bw.c` | resident watchdog, no forks, below 0.1% CPU |
| `brightness-early.sh` | early restore at boot |
| `com.local.brightness-early.plist` | LaunchDaemon for the above |
| `com.local.brightness-watch.plist` | LaunchAgent (KeepAlive) for `bw` |
| `install.sh` | builds what is missing, installs, fixes state-file ownership |

## Install

```bash
sudo bash install.sh
```

Then set a level; the watchdog stores user changes automatically.

```bash
/usr/local/bin/ds 0.4
```

State lives in `/Users/Shared/brightness.level`.

## Notes

- The state files in `/Users/Shared` must be owned by the logged-in user; a root-owned file silently breaks
  the watchdog, which then cannot write the last display state.
- The "save" role must not run at login before the restore, or it stores the pre-restore (100%) value.
- A shell version of the watchdog cost about 17 forks per second; the C version avoids all of them.
- These files are independent work offered under the MIT license, separate from the repository's
  PolyForm Noncommercial license.
