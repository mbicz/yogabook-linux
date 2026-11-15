# Battery charge limit (bq25890_charger)

`0001-power-supply-bq25890-add-charge-control-end-threshold.patch`
adds `POWER_SUPPLY_PROP_CHARGE_CONTROL_END_THRESHOLD` to the BQ25890 charger
driver, so the Yoga Book can stop charging at a chosen percentage instead of
sitting at 100% on AC indefinitely.

## Why this device needs it

The tablet is used on AC nearly all the time. After 109 cycles the gauge
(`bq27542-0`) reports 33.2 Wh of 37.3 Wh design capacity — 89.0% health — and
it sits pinned at 100%. Mainline `bq25890_charger.c` has no percentage-based
end-of-charge control: it exposes only voltage/current limits and `ONLINE`.

## Mechanism

The BQ25890 has no capacity comparator. The driver therefore reads the battery
gauge's `CAPACITY` and toggles `F_CHG_CFG` (REG03 bit 4):

- `F_CHG_CFG = 0` stops charge-FET conduction while VBUS still powers the
  machine. Register readback goes `0x1a → 0x0a`.
- `F_EN_HIZ` was rejected: it forces the system onto battery (discharge), which
  is not what a charge limit should do.

`threshold >= 100` means "no limit" and never inhibits, so a fresh boot at a
full battery does not leave the device permanently not charging.

Hysteresis avoids per-percent toggling: once inhibited, charging resumes when
`capacity <= threshold`; while charging, it inhibits at
`capacity >= threshold`.

Note what the limit does *not* do: clearing `F_CHG_CFG` disables the charge FET
while VBUS stays connected and carries the system load, so a battery already
above the limit stays where it is rather than discharging down to it. The
benefit is ending the sustained float at full charge, not enforcing a target
level. Reaching the limit from above needs the machine to run on battery.

The gauge registers as `bq27542-0` and does not appear in the charger's
`supplied_to` list, so the driver resolves it by `POWER_SUPPLY_TYPE_BATTERY`
via `power_supply_for_each_psy()` and watches it with a power-supply notifier.
`class_for_each_device()` stops on a non-zero callback return, hence the
`return 1` on success.

`bq25890_remove()` unregisters the notifier and, if charging was gated, writes
`F_CHG_CFG = 1`, so unloading the module cannot leave charging blocked.

## Verification (2026-10-31, on the device)

Kernel `6.18.58-yogabook8-lts-cam-thermal1`, module SHA-256
`cd83b850aea2b3b930f19aafbc641f194b57e7ab1bf646fa53dc5cc8eef6dc6d`.

| step | result |
| --- | --- |
| boot with default 100%, battery at 100% | `Full`, REG03 `0x1a` — no spurious inhibit |
| write 80 | `Not charging`, REG03 `0x0a`, dmesg `Suspended charging at 100% (limit 80%)` |
| write 100 | `Charging`, REG03 `0x1a`, dmesg `Resumed charging at 100% (limit 100%)` |

`/sys/class/power_supply/bq25890-charger-0/charge_control_end_threshold` is the
control; it accepts 0–100 and is writable.

The stock module is kept at `/root/bq25890-backup-20261031/bq25890_charger.ko.stock`
(SHA-256 `a0fcfe71…`).

## Persistence

`/etc/systemd/system/yogabook-charge-limit.service` (oneshot,
`RemainAfterExit=yes`) writes `80` at boot, guarded by
`ConditionPathExists` on the sysfs attribute so a kernel without the patch
skips it silently. It is enabled and active.

## Build

The module is built out-of-tree against the running kernel's config, in the
`yb-led-build` Lima VM (Debian 12 chroot at `/srv/bookworm`, no kernel builds on
the tablet itself):

```sh
make ARCH=x86_64 CROSS_COMPILE=x86_64-linux-gnu- \
     O=/work/kernel-out-yb8 LOCALVERSION=-yogabook8-lts-cam-thermal1 \
     M=/work/kernel-src-r2/drivers/power/supply modules
```

`O=` must point at the tree configured for `-yogabook8-lts-cam-thermal1`;
building against the yb7 out dir yields a vermagic the running kernel rejects.
Then install over
`/lib/modules/$(uname -r)/kernel/drivers/power/supply/bq25890_charger.ko` and
regenerate the initramfs with `mkinitramfs -o <new> $(uname -r)` followed by
`mv`, then reboot.

The module is loaded from the initramfs, and `bq25890_charger` cannot be
unloaded on this board: its refcount stays at 1 through `rmmod`, I2C unbind of
both `i2c-bq25892` and `i2c-bq27542`, and `usb_otg_vbus`/`vsys` inspection — a
device link pins it (`bq27542-0` reports `waiting_for_supplier`). Reboot is the
only way to apply a new build.

## Upstream

Not submitted. `torvalds/linux` master has no
`POWER_SUPPLY_PROP_CHARGE_CONTROL_END_THRESHOLD` user for BQ25890; the property
exists and is used by `wilco-charger.c` and `cros_charge-control.c`, so the
approach matches existing precedent.
