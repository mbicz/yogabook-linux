# Smoother automatic brightness (gnome-settings-daemon)

`0001-power-advance-ambient-brightness-independently-of-sensor-events.patch`
(SHA-256 `a49a6eed95120e79640d2c182acc5df45c54b353df90f7060e21dc4afd6c3d62`)
applies to gnome-settings-daemon 48.1 and to the current `gnome-48` branch.

## Defect

In 48.1 the ambient low-pass filter only runs inside `iio_proxy_changed()`, so
brightness advances once per ALS notification. The filter's alpha is derived
from the gap between notifications:

```
alpha = 1 / (1 + GSD_AMBIENT_TIME_CONSTANT / (now - last))
```

Two consequences on this device:

- When the ALS repeats or stops changing, no further notifications arrive and
  the ramp stalls part-way to the target (measured: 23% of the way to a 20%
  target, then nothing).
- After a quiet interval, the first new notification carries a large alpha and
  applies the whole new target at once. That is the visible multi-step jump
  the user reported on 2026-10-29 when the room light changed abruptly.

## Change

- A 50 ms `g_timeout_add` timer advances the same exponential filter, using a
  fixed alpha for the timer interval. Elapsed quiet time is no longer folded
  into a new target.
- `iio_proxy_changed()` only updates `ambient_target` and starts the timer.
- The timer settles exactly on the target instead of approaching the last whole
  percent forever, and stops itself.
- Pending transitions are cancelled on: `ambient-enabled` turned off, idle mode
  leaving normal, session deactivation, sensor release/vanish, and shutdown.
- Manual changes stay immediate: `StepUp`/`StepDown`/`Cycle` and the `Brightness`
  property stop the timer and are exempt while the write is in flight
  (`ambient_manual_pending`), then renormalise on the next reading.

## Verification (2026-10-30, on the device)

- `meson test` suite `gnome-settings-daemon`: 11/11 OK, including two new
  `test-power Ambient` scenarios (`PowerPluginTestAmbient`) that run the real
  daemon against mock logind/backlight/upower/mutter on private D-Bus buses.
- Against the unpatched 48.1 binary, `test_single_reading_converges_without_jumping`
  fails at "brightness did not reach the ambient target: 23" — the stalled ramp.
- Live greeter session: manual 50→70→50 moved the panel 129→179→129 immediately.
  The passive ALS reads 0.0/0.3 lux in a dark room, so the ambient ramp itself
  could not be exercised against real hardware without a controlled light
  stimulus; the ramp behaviour is covered by the mock-sensor tests above.

## Build and install

```sh
# on a trixie host with the source unpacked at /path/gnome-settings-daemon-48.1
patch -p1 < 0001-power-advance-ambient-brightness-independently-of-sensor-events.patch
# bump debian/changelog to 48.1-1.1+0yb1. The `+0yb1` suffix alone sorts
# BELOW a Debian revision like 48.1-1+deb13u1, so a security upload would
# silently replace the patched build; `-1.1` sorts above 48.1-1+deb13u1.
DEB_BUILD_OPTIONS="nocheck parallel=2" dpkg-buildpackage -b -us -uc
sudo apt-get install ./gnome-settings-daemon_48.1-1.1+0yb1_amd64.deb \
                     ./gnome-settings-daemon-common_48.1-1.1+0yb1_all.deb
```

`gsd-power` is `RefuseManualStop=yes`, so after the install send it SIGTERM; it
is respawned by gnome-session from `/etc/xdg/autostart`. Verify with
`sha256sum /proc/$(pgrep -u Debian-gdm -x gsd-power)/exe` against the packaged
binary.

Rollback: keep `gnome-settings-daemon_48.1-1_amd64.deb` and
`gnome-settings-daemon-common_48.1-1_all.deb` from `apt-get download`, then
`apt-get install --allow-downgrades` them.

## Upstream

Not submitted. The patch is against 48.1; upstream `main` has replaced this
path with `org.gnome.Shell.Brightness` and shell-side animation, so the defect
does not exist there in this form. `gnome-48` is the branch where it still
applies.
