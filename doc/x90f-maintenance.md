# YB1-X90F maintenance

Updated 2026-11-15; the device was last verified on 2026-11-03. This is the tested Android-edition X90F/Debian setup in this fork, not a claim of X91L compatibility or upstream acceptance. The project is paused; see the [README](../README.md).

## Current kernel

- [6.18.58-yb8.0 source](https://github.com/mbicz/yogabook-linux-kernel/tree/v6.18.58-yb8.0)
- [Image package, full config, build instructions and checksums](https://github.com/mbicz/yogabook-linux-kernel/releases/tag/v6.18.58-yb8.0)
- [Landlock patch on the preceding accepted build](https://github.com/mbicz/yogabook-linux-kernel/commit/3897343f7583d5260a0c4d5bf7063bad5b308b3f)

The change enables Landlock for Tracker/LocalSearch 3.8 content extraction. All other full-config values and all hardware patches are retained. The kernel booted on the device, exposed Landlock ABI 7 and indexed a new content-only search token that was absent before the reboot. It is now the accepted default.

A separate LocalSearch startup failure came from an old SQLite FTS view using an ambiguous `GROUP BY ROWID`. A complete cache backup and clone reproduction preceded a transactional correction to `GROUP BY "rdfs:Resource".ID`; the normal migration then completed. Tag counts and hashes were preserved. That one-off database repair is not an automated fix in the kernel package. Private databases and device logs are not published.

This branch pins the kernel submodule at `3897343f7` (`v6.18.58-yb8.0`) and fetches it from our fork, so a recursive checkout reproduces the kernel **source**. It does not reproduce the installed image exactly: the `bq25890` charge-limit patch ([`devices/YB1-X90/charge-limit`](../devices/YB1-X90/charge-limit)) is not in that branch and was applied to the installed module only.

## Working stack and evidence limits

- Halo keyboard/touchpad: [seven-commit submission](https://github.com/mbicz/chromiumos_touch_keyboard/compare/deccecb...118a3bef526bdeeb982bb8391b40178fb9027ade). Installed daemon scenarios cover keys, modifiers and release behavior; physical typing is a separate acceptance check.
- Tablet/posture handling: [support submission](https://github.com/mbicz/yogabook-support/compare/517d48a...6b0f9fba051fbe567eef54838bb08482f5d15b65), with generic kernel/uinput switches rather than a GNOME-only policy.
- Cameras: [yogabook-camera v0.1](https://github.com/mbicz/yogabook-camera/tree/v0.1) provides both PipeWire nodes, manual switching, screen/hinge orientation and rear autofocus. [KERNEL.md](https://github.com/mbicz/yogabook-camera/blob/v0.1/KERNEL.md) maps the kernel and ACPI prerequisites. Both real streams delivered 40 frames on this kernel. Daylight colour/clipping/focus quality and GUI/rotation acceptance remain separate.
- Audio: the existing RT5677/SST firmware and routing already work. Silent 48-kHz stereo playback passed on this kernel; this is not a fresh audible-speaker or microphone test, and this release adds no audio fix.
- Ambient light: a user-confirmed cover–bright–cover test produced 0–903.9 lux on both ALS interfaces, with reads below 100 ms. No ALS workaround was added. This establishes response, not calibration. Automatic brightness remains disabled; the older unexplained panel-brightness jump is not reproduced or fixed.
- GNSS: [Yoga-Book-GNSS](https://github.com/Yoga-Book/Yoga-Book-GNSS) 1.0.4 with the X90F patch in [`devices/YB1-X90/gnss`](../devices/YB1-X90/gnss), plus a stock-ROM runtime the owner supplies. On 2026-11-03 it produced a real 3D fix (11 satellites, HDOP 0.80, about 12 m). After three suspend cycles the receiver stayed cold indoors; recovery with sky view is untested. This replaces the earlier [bcm4752-init](https://github.com/mbicz/bcm4752-init/tree/79cb7f52a52ce5ec543a9933193878adde6458f4) experiment, which brings the receiver up but does not compute a position and is no longer deployed.
- Sleep: see [`devices/YB1-X90/sleep`](../devices/YB1-X90/sleep). Three RTC-woken s2idle cycles resumed cleanly on 2026-11-03 and the earlier repeat-suspend failure did not reproduce. A lid-close wake was never tested, and the device did not answer ping or SSH on 2026-11-06 or 2026-11-15, so sleep is **not accepted**.
- Charge limit: [`devices/YB1-X90/charge-limit`](../devices/YB1-X90/charge-limit) adds `charge_control_end_threshold` to `bq25890`; it is set to 80% and toggled Charging and Not charging on hardware.
- Ambient brightness ramp: [`devices/YB1-X90/gsd-ambient`](../devices/YB1-X90/gsd-ambient) is the local gnome-settings-daemon 48.1 patch, installed as 48.1-1.1+0yb1. Automatic brightness stays disabled.

The [21-topic kernel submission](https://github.com/mbicz/yogabook-linux-kernel/compare/bc6f91d9a5c3cd966f2ce7676b61495525394835...10aa1f4d0fc4ab42faca26fce07ab92d3d728783) targets jekhor's 6.17 line. It built but was not booted. It is not the same tree as the tested LTS release; runtime evidence must not be silently transferred between them.

## Boot and rollback

The accepted entry is `yogabook8-lts-landlock`, kernel release `6.18.58-yogabook8-lts-cam-thermal1`, build `#yb8.0`. Its saved default was changed only after the one-shot boot and runtime checks.

On this device, the preceding accepted entry remains `yogabook7-lts-6.18.58` / `#yb7.8`, and the older `yogabook7-lts` / `#yb7.7` is also retained. Revert the saved default with:

```sh
sudo grub-set-default yogabook7-lts-6.18.58
```

That command changes the next default; it does not reboot. Entry names are local to this installation. As of 2026-10-28 no UEFI variables, accepted boot arguments or `GRUB_SAVEDEFAULT` setting had changed; the sleep masks were removed on 2026-11-03 (see Sleep above).

`/boot` was reclaimed on 2026-10-31 (98% to 31% full, about 300 MB free). Before another image install, back up the boot state, check capacity and retire only owned superseded files, not the running image or verified rollback. This image depends on existing early ACPI overrides and firmware; it is not a drop-in installer for an unconfigured tablet.

## Maintenance decision

Keep the working 6.18 LTS stack and review 6.18.y updates. Do not freeze at 6.18.58, but do not replace the hardware stack just to match another project's version numbers. [Kernel.org](https://www.kernel.org/releases.html) projects 6.18 support through December 2028, subject to revision. At this check, 6.18.59 is available but not installed. The custom patches and packages remain our responsibility; Debian APT does not rebuild this kernel automatically.

The [pinned comparison with Yoga-Book](x90f-yoga-book-comparison.md) records the overlap and different approaches. Their X91L/Ubuntu work offers useful collaboration and selective-backport candidates, not a demonstrated reason to replace this X90F/Debian deployment.

## Project state

Paused on 2026-11-15 as stable; nothing is being upstreamed. The reasons that would justify resuming, including when to rebase on the Yoga-Book organization instead of this line, are in the [README](../README.md#where-this-sits-relative-to-other-projects). The first step on resuming is to find out why the tablet did not answer the network after the 2026-11-03 sleep test.
