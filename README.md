# Yoga Book Linux: YB1-X90F fork

This is mbicz's fork of [jekhor/yogabook-linux](https://github.com/jekhor/yogabook-linux).
**Read the `yb-release` branch.** The fork's `master` is an unchanged mirror of
jekhor's umbrella and is not maintained.

`yb-release` documents and pins a tested setup for the **Android-edition
Lenovo Yoga Book YB1-X90F** (YETI-11 firmware) running **Debian 13 (trixie)** on a
custom **6.18.58 LTS** kernel. It is not a claim of YB1-X91L compatibility.

> **Project state, 2026-11-15: stable and paused.** Nothing here is being sent
> upstream. The last full device check was 2026-11-03; the tablet did not answer
> ping or SSH when probed on 2026-11-06 and 2026-11-15 (see [Sleep](#what-works-and-what-does-not)).

## Where this sits relative to other projects

| Project | Role | State, checked 2026-11-15 |
|---|---|---|
| [jekhor/yogabook-linux](https://github.com/jekhor/yogabook-linux) and its component repositories | Original umbrella; parents of our forks | Quiet. Umbrella last commit 2026-04-07; kernel `master` (6.17.4) 2026-01-23; keyboard 2026-02-03; support 2025-11-02. A reply on our issue [#78](https://github.com/jekhor/yogabook-linux/issues/78) (closed 2026-10-28) from DerKnerd, who publishes the community packages, pointed to the Yoga-Book organization as the more recent project. |
| [Yoga-Book organization](https://github.com/Yoga-Book) | Active successor; targets the **YB1-X91L** on Ubuntu | Kernel is a 7.2-based tree, last pushed 2026-09-05; GNSS and Camera 2026-08-31; Halo-Keyboard 2026-09-26. Issues are disabled on its kernel repository. None of the latest 60 kernel commits mention X90, YETI or Android. |
| this fork | **YB1-X90F** on Debian 13 | See below. |

Prepared contributions exist, **none opened**: 21 kernel patches against jekhor's
6.17.4 line ([`jekhor-submission`](https://github.com/mbicz/yogabook-linux-kernel/compare/bc6f91d9a5c3cd966f2ce7676b61495525394835...10aa1f4d0fc4ab42faca26fce07ab92d3d728783)),
7 keyboard commits and 2 support commits.

**Why not rebase on the Yoga-Book organization?** There is no strong reason today.
Their work covers the X91L, their camera needs raw Bayer data that this kernel's
ISP path does not expose, and nothing in their recent history touches the X90F.
Revisit if any of these happens:

1. They add X90F/YETI-11 support, or say they will accept patches for it.
2. A fix we need exists only on their tree and cannot reasonably be backported.
3. Raw Bayer capture becomes available on our 6.18 line, so their userspace
   camera can run here. That needs one kernel capability, not their whole tree.
4. Our 6.18 LTS line nears end of life (projected December 2028).

A newer version number alone is not a reason.

## Tested stack

| Component | Version | Where it comes from |
|---|---|---|
| Kernel | `6.18.58-yogabook8-lts-cam-thermal1` (`v6.18.58-yb8.0`) | [release](https://github.com/mbicz/yogabook-linux-kernel/releases/tag/v6.18.58-yb8.0): `.deb`, full config, build notes, checksums |
| Charge limit | out-of-tree `bq25890` module | patch in [`devices/YB1-X90/charge-limit`](devices/YB1-X90/charge-limit); **not** in the published kernel branch |
| OS | Debian 13 (trixie), GNOME 48 on Wayland | |
| Halo keyboard | `touch-keyboard` 1.5.0+yb6 | submodule `touch-keyboard` |
| Posture, modes handler | `yogabook-support` 1.6.1+yb1 | submodule `yogabook-support` |
| Cameras | `yogabook-camera`; the source is identical to the deployed build | [mbicz/yogabook-camera](https://github.com/mbicz/yogabook-camera) v0.1 |
| GNSS | `yogabook-gnss` 1.0.4+yb1 | [Yoga-Book-GNSS](https://github.com/Yoga-Book/Yoga-Book-GNSS) plus [`devices/YB1-X90/gnss`](devices/YB1-X90/gnss); you supply the runtime |
| Ambient brightness | `gnome-settings-daemon` 48.1-1.1+0yb1 | patch in [`devices/YB1-X90/gsd-ambient`](devices/YB1-X90/gsd-ambient) |
| Sleep | lid-watch service and drop-ins | [`devices/YB1-X90/sleep`](devices/YB1-X90/sleep) |
| Audio | RT5677 / SST, UCM profile | submodule `alsa-ucm-conf-yogabook` |

## What works and what does not

Last verified on the device on **2026-11-03**, kernel `6.18.58-yb8.0`. "Works" means
observed on the device; the evidence column says what was and was not checked.

| Area | Status | Evidence and limits |
|---|---|---|
| Display and backlight | Works | 1200×1920 panel; backlight 0, 128 and 255 each read back exactly. Automatic brightness is off by choice. |
| Halo keyboard and touchpad | Works | Daemon active; virtual keyboard and touchpad present. Physical typing is a separate human check on each boot. |
| Front touchscreen | Works | HiDeep device enumerates as a touch device. |
| Pen | Partial | Wacom 056A:0169 enumerates as a tablet. Pen direction and rotation mapping were never confirmed by hand. |
| Rotation and tablet mode | Works | Kernel tablet-mode switch plus a posture switch (on when folded or hinge ≥165°, off at 30 to 155°) drive GNOME auto-rotation. Verified headless. |
| Cameras | Partial | Both PipeWire cameras streamed 40 frames each on this kernel (2026-10-28), with manual switching and screen/hinge orientation; on 2026-11-03 both nodes were present and the front sensor enumerated. Rear autofocus is contrast-based because the AD5823 lens driver is not instantiated. **Colour and sharpness have never been human-accepted.** |
| Audio | Partial | Card, speaker sink, digital microphone source and headset-jack device are present; a test tone plays through the sink without error. Audible quality, microphone capture and jack detection were not re-checked. |
| WiFi and Bluetooth | Works | WiFi connected at −50 dBm, 144 Mbit/s; Bluetooth controller powered. |
| Battery and charge limit | Works | Gauge reads correctly; an 80% limit toggled Charging and Not charging on hardware. A battery already above the limit is not discharged. Needs the patch above. |
| Sensors | Works | Ambient light responds to light (calibration not established); hinge angle, accelerometer and iio-sensor-proxy orientation read live. |
| Haptics | Partial | Two drv2604l devices report force-feedback support. How they feel was not checked. |
| Thermal | Works | Zones 20 to 37 °C during a 20 s two-core load; `step_wise`. |
| GNSS | Partial | A real 3D fix on 2026-11-03 (11 satellites, HDOP 0.80, about 12 m) before the first suspend. After three suspend cycles the receiver stayed cold indoors (2 to 3 satellites, no fix) through restart, rfkill cycle and reboot. Needs a stock-ROM runtime you supply. |
| Sleep (s2idle) | **Not accepted** | Three consecutive suspends with an RTC wake all resumed cleanly on 2026-11-03, and the earlier repeat-suspend failure did not reproduce. A real lid close and reopen was never tested. The tablet has not answered ping or SSH on 2026-11-06 and 2026-11-15; the cause is unknown (asleep, off or discharged). |
| LTE modem | n/a | The X90F has none. |

## Using this branch

```sh
git clone -b yb-release https://github.com/mbicz/yogabook-linux.git
cd yogabook-linux
git submodule update --init touch-keyboard yogabook-support yogabook-hardware-check alsa-ucm-conf-yogabook
```

Every submodule now resolves from a public URL, and each pin was fetched
anonymously before it was recorded. Adding `yogabook-linux-kernel` clones the
Linux kernel, which is large. Prefer the release image over building from source.

The kernel image is **not a drop-in installer**. It relies on early ACPI overrides
and firmware that must already be configured, and `/boot` is small; read
[X90F maintenance](doc/x90f-maintenance.md) for the boot, rollback and capacity
notes first. See also the [comparison with Yoga-Book](doc/x90f-yoga-book-comparison.md).

## Kernel-upgrade hardware verification

The `yogabook-hardware-check` submodule provides a small generic pytest framework
and a separate YB1-X90F suite. Checks are ordinary Python assertions or runnable
shell files; board-specific options stay with the suite, not in the framework.

```sh
git submodule update --init yogabook-hardware-check
cd yogabook-hardware-check
python3 -m venv .venv
.venv/bin/python -m pip install -e .
.venv/bin/yb-check list checks/yogabook
sudo .venv/bin/yb-check run checks/yogabook --report reports/kernel.json -v
```

Enumeration, headless operation, userspace integration and physical observations
are independent results. Pending human checks never erase a successful low-level
assertion, and enumeration alone never establishes operation. Camera, audio and
GPS exercise is opt-in; missing prerequisites are blocked, not passed. See the
[suite guide](yogabook-hardware-check/checks/yogabook/README.md). Keep reports and
proprietary runtime data out of git.

---

# Inherited upstream material (YB1-X91 / Windows edition)

Everything below is jekhor's original text. It describes the Windows-edition
models and is **not** the acceptance report for this fork; for the X90F, use the
sections above. In particular its "Cameras" exception and "Camera doesn't work"
note do not describe the X90F setup here.

## Status

Instructions are for YB1-X91 (Windows) models. Android (YB1-X90) version should works
also, with some exception (check the Issues). To boot from USB, enable USB host and
disable Secure Boot as described here:
https://web.archive.org/web/20220516142318/https://poz1.com/windows-on-android-lenovo-yogabook/

Most of the hardware is functional, with the following exceptions:
- Cameras
- LTE modem (in L modifications of the tablet)

Most Linux patches have been mainlined, except for the sound drivers.

## What You Need to Know

The Yoga Book tablet features a sensor keyboard, which essentially functions as
a touchpad. To use it as a keyboard, a userspace 'driver' is utilized.
Therefore, installing the OS without connecting a physical keyboard via an OTG
hub can be challenging.

## Quick Start

I personally use Debian, so the instructions below describe its installation.
Ubuntu should also be compatible. For other distributions, you may need to
download and build packages from the source.

Instructions are approximate, details may vary. Please create an issue if
something goes wrong.

### Repositories

If you want to use a prepackaged repository, there are repositories available for Arch Linux
(and derivatives), Debian (and derivatives), Ubuntu (and derivatives) and Alpine Linux (and derivatives).

Check the Wiki on how to install it using the repositories on your system. Currently supported are:

- Ubuntu 24.04, 25.10 and 26.04
- Debian Trixie, Forky, Testing and Sid
- Arch Linux
- Alpine Linux

Check these links for the build scripts:

- [Debian/Ubuntu](https://gitlab.imanuel.dev/packages/yogabook/building/apt-build)
- [Arch Linux](https://gitlab.imanuel.dev/packages/yogabook/building/pacman-build)
- [Alpine Linux](https://gitlab.imanuel.dev/packages/yogabook/building/apk-build)

The package repo is found [here](https://packages.ulbricht.casa/linux-yogabook/-/packages).

If you want to use a different distro like Fedora, please comment
[this](https://github.com/jekhor/yogabook-linux/issues/54) issue.

### OS Installation

1. Download an ISO image and write it to a USB flash drive (refer to
   [Debian's official site](https://debian.org/) for instructions).
2. Connect a powered microUSB OTG hub to the tablet (non-powered may works also, let's try).
3. Connect the USB flash drive and a keyboard to the hub.
4. Power on the tablet while holding the Volume Up key.
5. Select the USB drive in the boot menu.
6. Install Debian as you normally would, and boot into the new system. The
   Gnome desktop is recommended because it supports screen rotation based on
   accelerometer data.
7. Download the latest `.deb` packages (check the Tags section in this repository):
   - `linux-image`
   - `touch-keyboard`
   - `yogabook-support`
   - `alsa-ucm-conf-yogabook`
8. Open a terminal emulator, navigate to the directory containing the `.deb`
   files, and install them using the command `sudo dpkg -i --force-depends *.deb`.
10. Resolve any missing dependencies with the command `sudo apt -f install`.
11. For Debian, install firmware packages also: `sudo apt install firmware-intel-sound firmware-brcm80211`.
12. Reboot. Verify that the new kernel is in use with the command `uname -a`
    (the output should include 'yogabook' in the version).
13. If the system has a newer kernel than the one you downloaded, reboot again,
    select the Yoga Book kernel from the boot menu, and verify that it boots
    successfully. After this, remove the default kernel from the system with
    `sudo apt remove linux-image-<version>` command.
14. The touch keyboard should now be operational, allowing you to detach
    the OTG hub and enjoy your Yoga Book.

## Troubleshooting

Known issues:
- Camera doesn't work.
- To have automatic screen rotation working, it should be supported by desktop
  environment. Try Gnome first.
- My Yoga Book randomly freezes after some time. If you have the same behavior,
  let me know.
