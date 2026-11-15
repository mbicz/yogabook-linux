# Sleep (s2idle) on the YB1-X90F

The files here make one component the only owner of lid-close suspend.

| File | Install as | Purpose |
|---|---|---|
| `yogabook-lid-watch` | `/usr/local/sbin/yogabook-lid-watch` | Reads `SW_LID` from the `gpio-keys` input device (needs `python3-evdev`), arms `/sys/power/wakeup_count`, then runs `systemctl suspend`. A wake event that arrives first cancels the suspend. |
| `yogabook-lid-watch.service` | `/etc/systemd/system/` | Runs the watcher. |
| `logind-yogabook-lid.conf` | `/etc/systemd/logind.conf.d/yogabook-lid.conf` | logind ignores the lid switch so there is a single suspend owner. |
| `logind-yogabook-nosleep.conf` | `/etc/systemd/logind.conf.d/yogabook-nosleep.conf` | Ignores idle, suspend key and short power-key presses; long power press still powers off. |
| `yogabook-sleep.conf` | `/etc/systemd/sleep.conf.d/yogabook.conf` | `SuspendState=mem`, so a wake event cancels suspend instead of falling back to `freeze`. |

`mem_sleep` on this board reads `[s2idle]`, so `mem` is s2idle. Valid
`/sys/power/state` values are `freeze`, `mem` and `disk`; writing `s2idle`
fails with an I/O error.

## History

On 2026-10-18 a logind suspend request fired with the lid shut during a remote
session and took the network down. Every sleep path was then masked
(`sleep.target`, `suspend.target`, `hibernate.target`, `hybrid-sleep.target`)
until the resume path could be trusted. A search of all 90 retained kernel
journals on 2026-11-03 found no `PM: suspend entry` at all, so the suspend
path had never been exercised on this configuration.

## Evidence, 2026-11-03 (kernel 6.18.58-yb8.0)

Targets unmasked, `yogabook-lid-watch` enabled and ready on `gpio-keys`. Three
consecutive suspends, each with an RTC wake alarm armed beforehand:

| Cycle | Asleep |
|---|---|
| 1 | 91 s |
| 2 | 61 s |
| 3 | 46 s |

Each logged `PM: suspend entry (s2idle)` then `PM: suspend exit`. Afterwards no
unit had failed, and input, sensors, camera nodes, audio, WiFi and the charge
limit were intact. The repeat-suspend failure seen earlier did not reproduce.

## Not established

- A real lid close and reopen has not been tested; only RTC wakes were used.
- The GNSS receiver comes back cold after resume (see `../gnss/README.md`).
- When probed on 2026-11-06 and 2026-11-15 the device did not answer ping or SSH.
  Nothing was checked in between. The cause is unknown: it may be asleep,
  switched off or discharged. Treat sleep as not accepted.

## Test safely

Arm an RTC wake first, so a stuck suspend cannot strand a remote machine:

```sh
sudo sh -c 'echo 0 > /sys/class/rtc/rtc0/wakealarm; echo $(( $(date +%s) + 90 )) > /sys/class/rtc/rtc0/wakealarm; echo mem > /sys/power/state'
```
