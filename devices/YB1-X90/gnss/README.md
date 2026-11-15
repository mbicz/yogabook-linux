# GNSS on the YB1-X90F

The X90F's BCM4752 receiver runs through the community
[Yoga-Book/Yoga-Book-GNSS](https://github.com/Yoga-Book/Yoga-Book-GNSS)
package with one patch from this directory. Nothing proprietary is included.

- Base: upstream `6a2c6d78940d7d46c7a70e4d5ecfa7e40649edaa` ("debian: release 1.0.4").
- Result: `yogabook-gnss 1.0.4+yb1`.
- Patch: [0001-yogabook-gnss-x90f-machine-profile.patch](0001-yogabook-gnss-x90f-machine-profile.patch)

## Why a patch is needed

Upstream targets the YB1-X91L and hard-codes it: DMI `LENOVO` / `Lenovo YB1-X91L`
and `/dev/ttyS5`. This X90F reports Intel DMI strings (`Intel Corporation`,
`CHERRYVIEW D1 PLATFORM`, board `Cherry Trail Tablet`, version `YETI-11`) and
exposes the same receiver as `/dev/ttyS2`. Both machines reach it through the
ACPI UART `8086228A:01`.

The patch keeps the safety checks rather than removing them:

- a machine-profile table (`YB1-X90F`, `YB1-X91L`) selected from DMI at install
  time and stored in `/etc/yogabook-gnss/machine`;
- the launcher and health check read the UART from that profile instead of
  assuming `ttyS5`;
- the udev rule matches the UART by PCI function `8086228A:01`;
- the stock ROM build id and fingerprint stay pinned to the verified X90F image.

## Build

```sh
git clone https://github.com/Yoga-Book/Yoga-Book-GNSS.git
cd Yoga-Book-GNSS
git checkout 6a2c6d78940d7d46c7a70e4d5ecfa7e40649edaa
git apply /path/to/0001-yogabook-gnss-x90f-machine-profile.patch
python3 -m unittest discover -s tests -p 'test_*.py'
bash tests/test-shell.sh
dpkg-buildpackage --no-sign -b
```

The patch applies cleanly to that commit; the 6 Python tests and the shell policy
checks pass.

## Runtime (not in this repository)

The receiver's position solution is computed by Broadcom's Android daemon, which
this project does not redistribute. You must supply a runtime from your own
device's stock ROM and import it:

| Path in the archive | Source |
|---|---|
| `system/build.prop` | stock ROM |
| `system/bin/linker64` | stock ROM |
| `system/lib64/*.so` | stock ROM (25 libraries were required) |
| `system/vendor/bin/gpsd` | the stock `glgps_intelx86android` daemon, relocated; SHA-256 `d135fc515dd3802dcd9fe807b942e1e112767f56b5a44463c64121317753c715` |
| `system/vendor/etc/gps.xml` | the stock XML configuration |

`gps.conf` and `gps.cer` were absent from the retained tree and the daemon ran
without them, so the importer treats them as optional. That shows this build
ran without them, not that they are never used.

```sh
sudo yogabook-gnss-import runtime.tar
```

gpsd must be the only reader of the vendor NMEA FIFO. Leave `DEVICES=""` in
`/etc/default/gpsd`: `/dev/ttyS2` carries binary GLME, which gpsd cannot parse.

## Evidence and limits

- 2026-11-03: a real 3D fix: 11 satellites used, HDOP 0.80, about 12 m error,
  GGA timestamps advancing with UTC. `yogabook-gnss-health` reported PASS.
- After three suspend/resume cycles the same day the receiver stayed cold
  (2 to 3 satellites tracked, no fix) through a service restart, an rfkill
  cycle and a reboot. Those tests were indoors; whether it recovers with sky
  view was not tested.
- The patch was reconstructed from the shell history that built the installed
  package, then verified by applying it to a fresh checkout of the base commit
  and running the tests. It has not been byte-compared with the installed files.
