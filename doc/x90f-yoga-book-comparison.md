# Yoga-Book comparison and production-kernel decision

Recorded 2026-10-28. Source/history comparison, not a new hardware test.

## Decision: retain the working X90F stack

Keep the custom 6.18 LTS kernel, Debian userspace, kernel-ISP camera path and
existing SST audio. Do not migrate to the Yoga-Book organization's kernel or
userspace packages merely because they are newer or more extensively packaged.
Upstream collaboration and production deployment are separate decisions.

Keep taking reviewed 6.18.y stable updates; do not freeze the installed 6.18.58
image indefinitely. Kernel.org currently projects **December 2028** as 6.18's
end of life, subject to revision. Its release metadata lists **6.18.59**, released
2026-10-26, as the latest 6.18 update when checked. No update was installed during
this comparison.

- [Official longterm support schedule](https://www.kernel.org/releases.html).
- [Official release metadata](https://www.kernel.org/releases.json).
- Accepted device source: [6.18.58, fe92d01241f2adc9acec9e78dc232c0a21f8608e](https://github.com/mbicz/yogabook-linux-kernel/tree/fe92d01241f2adc9acec9e78dc232c0a21f8608e).
- [Deployment and rollback evidence](x90f-maintenance.md#boot-and-rollback).

LTS support applies to the upstream baseline. It does not maintain our added
AtomISP algorithms, board quirks, camera daemon, ACPI overrides or packaging.
Those remain our responsibility. Stable updates can still conflict with local
changes or cause hardware regressions; review and validate each candidate.
Debian security updates do not automatically rebuild this custom kernel.

Use the existing maintenance approach: preserve a known-good boot entry, review
the stable delta against our customizations, build a candidate, verify boot and
initramfs prerequisites, and exercise the changed paths before promotion. The
previous 6.18.57-to-6.18.58 update retained all 48 customized source paths with no
overlap in that stable delta; that is evidence for that update, not a promise
about later releases.

Consider a different kernel line only for a demonstrated benefit: a required
fix that cannot reasonably be backported, materially improved suspend/power or
hardware operation, upstream replacement of enough local code to reduce actual
maintenance, a userspace compatibility requirement, or approaching LTS EOL.
Neither a newer version number nor another project's feature list establishes
such a benefit. Reliable repeated suspend, intermittent ALS and remaining
optical/peripheral acceptance are still open on our stack.

## Scope and inspected revisions

The organization primarily integrates the Windows-edition YB1-X91L on Ubuntu.
Our device is the Android-edition YB1-X90F on Debian, with Intel/YETI-11 firmware.
Shared silicon does not imply identical ACPI identities, wiring, clocks, UART
numbering, DMI or power-control paths.

[DerKnerd referred issue #78 to the organization](https://github.com/jekhor/yogabook-linux/issues/78#issuecomment-6456848172).
The inspected authored changes identify Maurizio Casciano; some audio changes
credit jekhor. This establishes a related parallel effort, not proof of a formal
transfer of maintainership or incorporation of our recent patches.

Their inspected refs:

- Kernel product `Yoga-Book-YB1-X91L`: `8c290334c5a0af7cba5ba3d1f8da18c33a9117d8`.
  [Exact device delta](https://github.com/Yoga-Book/Yoga-Book-Linux-Kernel/compare/73ae59e975966d24e32926247ddb45a537ebe184...8c290334c5a0af7cba5ba3d1f8da18c33a9117d8):
  **24 commits**, zero behind the exact upstream predecessor. Comparisons against
  other mainline refs include unrelated upstream history; do not count those as
  Yoga Book patches. The tree's Makefile says 7.2.0 while ancestry includes the
  7.3 merge window; it is not simply a pristine stable 7.2 release.
- Kernel build: `17ee41996f9526aa18979d2d8ac59d9e9640bc5f`; product, build and
  submission branches diverge. Default `master` is not the device branch.
- Latest inspected subsystem submissions: media-v7 `bbd81ef43dfb8ad92572cdfb51ab2781e9a4cd02`,
  drv260x-v7 `b3bd39de6a7d9ffe12d7f9a2ce9f0d5dc646c21f`, platform-v3
  `321bb4680cb0ff74814292d3e8acfa66c8532577`, ASoC-v7
  `a500fec1dcd80e9126c72a3319447871bcd426c7`.
- Halo-Keyboard: `3a3de7ae79e42e31c9df5572a41e894ccd1f5e6b`.
- Sensors: `83511481a6219b1b496a0eb1e3e714cc0f2e3ecd`.
- Camera: `15fc3e5ec06900ec99773c5724027d9920783bd4`.
- GNSS: `6a2c6d78940d7d46c7a70e4d5ecfa7e40649edaa`.
- Validator: `bdf681dfc96d397dff8f2335a8521024864ad176`.
- UCM main: `3a1a2a2c5c138fd331bd6bed4820bbbfdfdd802b`; SOF topology release
  `yogabook-topology-v1.0.1`: `1a207703dc76af2b0d2c06c1fbc055f959e04de1`.
- Ubuntu-Autoinstall main: `a50aebda2171f985aaa168aa6988efb0a7381ba1`; ISO release
  `ubuntu-26.04.1-yb1-x91l-2026.09.02`: `dca6a337f6c7337c0a63b24606da7054ee885bf7`.
  Main and the released ISO pin different kernels; neither equals current product.

Our comparison anchors are the 21-topic kernel submission at
`10aa1f4d0fc4ab42faca26fce07ab92d3d728783`, deployed LTS above, seven keyboard
commits ending `118a3bef526bdeeb982bb8391b40178fb9027ade`, two support commits
ending `6b0f9fba051fbe567eef54838bb08482f5d15b65`, supporting-source snapshot
`ef322c30c4eb4e15de424650c0c7275fb074280c`, camera v0.1
`38610c6f5ba5f27f0cfcdc51b6c78255dfff9141`, bcm4752-init
`79cb7f52a52ce5ec543a9933193878adde6458f4`, and hardware-check
`1bb73fd752378e067aa20e525986b4e725651d25`.

## Camera kernel and userspace

Our path uses sensor OTP correction and AtomISP hardware image processing with
in-kernel AE/AWB/tone handling, then YU12 capture and native I420 PipeWire sources.
Their path captures raw Bayer and performs userspace demosaic, exposure/white
balance and CPU/GPU processing, then publishes V4L2 loopbacks. These are alternate
architectures, not successive versions of one daemon. Both serialize the single
physical capture engine.

Mapping all 21 topics in our kernel submission:

- **01, 03, 04, 05, 17:** our automatic ISP loop, anti-aliasing defaults,
  black-level/colour correction, noise-strength parameter and sensor-specific
  AWB/CCM have no equivalent in their inspected product delta. Their raw path
  avoids much of the processed ISP pipeline; this does not prove either path
  has better optical quality or efficiency on the other model.
- **02, 18, 19:** padding/geometry work overlaps but differs by branch and output
  format. Their processed-output metadata still differs from our full-range
  BT.601 correction. Their raw full-resolution mode does not replace our optional
  native NV12 preview capability. Our current daemon does not expose the old
  experimental full-resolution rear node.
- **06, 08:** our checksum-validated OTP lens-shading application is not duplicated.
  It may help either architecture after checking module data and avoiding double
  correction with userspace shading.
- **07, 09, 10, 11:** sensor/clock/bridge work overlaps in purpose, but X90 uses
  INT33FB/INT33BE and a two-lane rear configuration; their X91 uses
  OVTI2740/INT3477 and four rear lanes. Front links are configured differently.
  Android camera-client allowance and missing-clock/ACPI corrections remain
  distinct. Do not transplant mode tables or DMI matches indiscriminately.
- **16:** both lens drivers use WV517-family absolute focus. Ours adds the X90
  enable-gate description and gradual parking. Their userspace documentation
  records unresolved actuator provenance/maintainer questions. Our investigation
  identified Lenovo's published open-source driver as the register reference;
  provide that provenance when discussing convergence, rather than loading two
  competing drivers.
- **12, 13, 14, 15, 20, 21:** our LP855X configuration, edge-LED policy, EFI
  power-off, physical tablet switch, duplicate GPS/GPO4 quirks and YETI-11 i915
  backlight correction are not superseded by their product delta. Some are
  Android-firmware-specific; existing LP8557 support itself is inherited work.

Our daemon supplies Mutter/hinge orientation and rear preview contrast autofocus.
Their inspected preview has neither equivalent orientation handling nor a
preview AF loop, but their separate still helper implements full-resolution JPEG
capture and focus scanning. Lens search ranges are module-specific. Their
camera service also implements explicit resource ceilings and thermal gating;
ours already stops physical capture when idle. No performance or quality winner
was established by this source review.

Sources: [their kernel contract](https://github.com/Yoga-Book/Yoga-Book-Camera/blob/15fc3e5ec06900ec99773c5724027d9920783bd4/docs/KERNEL-CONTRACT.md),
[their processor](https://github.com/Yoga-Book/Yoga-Book-Camera/blob/15fc3e5ec06900ec99773c5724027d9920783bd4/src/yogabook_camera.py),
[their still helper](https://github.com/Yoga-Book/Yoga-Book-Camera/blob/15fc3e5ec06900ec99773c5724027d9920783bd4/src/yogabook_camera_capture.py),
[our kernel contract and exact topic links](https://github.com/mbicz/yogabook-camera/blob/v0.1/KERNEL.md).

## Keyboard, pen and posture

Both keyboard engines derive from ChromiumOS/jekhor. Their independent-history
Halo project credits that ancestry and adds dedicated worker supervision,
configuration/event validation, haptic error handling and service hardening.
Our late native-slot startup/gating fixes, live-contact Fn derivation, hinge
keyboard gate, printed-icon mappings and F10 cycling are not present there.
Their Fn handling already covers some rejection/lift cases; absence of our
algorithm is not proof that every old bug reproduces in theirs. Their boot-time
minimum-backlight helper is not equivalent to F10 cycling.

Both use integrated libwacom metadata rather than a fixed pen rotation matrix.
Our missing-I2C-binding recovery remains a board-specific workaround. Our support
handler publishes generic hall/hinge SW_TABLET_MODE posture; their separate Mutter
patch keeps native-portrait touchscreen panels sensor-managed outside tablet mode.
The compositor policy and generic posture interface are related, not equivalent.
Do not install duplicate descriptors, competing input daemons or posture switches.

Sources: [their input tree](https://github.com/Yoga-Book/Halo-Keyboard/tree/3a3de7ae79e42e31c9df5572a41e894ccd1f5e6b),
[our contact fix](https://github.com/mbicz/chromiumos_touch_keyboard/commit/6e597fdc2f7373ccd4a079772364f2eaa5e95df6),
[our F10 change](https://github.com/mbicz/chromiumos_touch_keyboard/commit/118a3bef526bdeeb982bb8391b40178fb9027ade),
[our posture handler](https://github.com/mbicz/yogabook-support/blob/6b0f9fba051fbe567eef54838bb08482f5d15b65/yogabook-modes-handler),
[their Mutter patch](https://github.com/Yoga-Book/GNOME-Mutter/commit/cc11a4c78df6aabd4a2b6cb2818546fef2a2cabe).

## Sensors and thermals

Their accelerometer rules still use allocated platform instance numbers; ours
identify physical HID hubs after observing renumbering. Port the identification
approach only after verifying X91 hub identities/mounting. Their X91 proximity
threshold metadata is additional work, not an established X90 setting.

Their thermald generator bypasses invalid STR0/STR2 and binds other live sensors
and cooling devices. Our extra LTS PMIC ADC patch and guarded ACPI corrections
restore those readouts. Measurement repair and cooling policy are complementary.
Neither establishes chassis calibration, cooling effectiveness or trip safety.
Their thermald drop-in replaces the service command globally while its generator
rejects non-X91L hardware; installing it on X90F as-is can stop thermald starting.

Sources: [their sensor rules](https://github.com/Yoga-Book/Yoga-Book-Sensors/blob/83511481a6219b1b496a0eb1e3e714cc0f2e3ecd/udev/61-yogabook-sensors.rules),
[their policy generator](https://github.com/Yoga-Book/Yoga-Book-Sensors/blob/83511481a6219b1b496a0eb1e3e714cc0f2e3ecd/libexec/generate-thermald-config.sh),
[their unsupported thermal backend](https://github.com/Yoga-Book/Yoga-Book-Linux-Kernel/blob/8c290334c5a0af7cba5ba3d1f8da18c33a9117d8/drivers/acpi/pmic/intel_pmic_chtwc.c#L260-L273),
[our ADC patch](https://github.com/mbicz/yogabook-linux-kernel/commit/c81f87c2f967ef24c6f3ddbf20d3611b41751a1e),
[our supporting sources](https://github.com/mbicz/yogabook-support/tree/ef322c30c4eb4e15de424650c0c7275fb074280c/contrib/x90f).

## Additional kernel fixes worth reviewing

- [DRV260x lifecycle/PM](https://github.com/Yoga-Book/Yoga-Book-Linux-Kernel/commit/b3bd39de6a7d9ffe12d7f9a2ce9f0d5dc646c21f):
  work quiescing, regulator accounting, configuration recovery and teardown.
  Worth selective LTS review; not proof of fixing our repeated-suspend failure.
- [Power-supply supplied-from handling](https://github.com/Yoga-Book/Yoga-Book-Linux-Kernel/commit/d1b9ffba81dbaeb32ce3181860b66e66449549a9):
  generic CONFIG_OF interaction with supplier notifications. They report restored
  charger callbacks. Our accepted 6.18.58 configuration has CONFIG_OF disabled,
  so this CONFIG_OF=y failure does not explain a fault on our current build.
  Do not copy current limits as an untested policy change.
- X91L XMM7260 power and SSIC restoration are cellular-modem work, not missing
  functionality on our Wi-Fi-only X90F and not BCM4752 GPS initialization.

## GPS

Their GNSS package executes the same proprietary S100043 Android/Bionic daemon
used by our vendor-runtime comparison tools. It adds validated runtime import,
persistent systemd operation, a Linux gpsd FIFO bridge, AppArmor integration and
transport-versus-fix health checks. It does not implement independent GLME
tracking. Our independent C initializer handles boot/patch/GLME initialization
only and cannot replace that tracking daemon while preserving positioning.

Their ttyS5/X91L service, udev and DMI assumptions differ from our ttyS2/X90F.
Our direct-image extraction, dependency-closure discovery, UART ownership checks
and bounded evidence capture complement their persistent packaging. A working
transport or FIFO is not proof of a geographic fix or desktop location delivery.

Sources: [their GNSS tree](https://github.com/Yoga-Book/Yoga-Book-GNSS/tree/6a2c6d78940d7d46c7a70e4d5ecfa7e40649edaa),
[our initializer](https://github.com/mbicz/bcm4752-init/tree/79cb7f52a52ce5ec543a9933193878adde6458f4),
[our vendor-runtime tools](https://github.com/mbicz/yogabook-support/tree/ef322c30c4eb4e15de424650c0c7275fb074280c/contrib/x90f/gps).

## Audio, installation and validation

Their jekhor-derived RT5677 machine-driver updates, SOF topology and conditional
SST/SOF UCM handling are substantive. Our SST audio already works with existing
DSP firmware and routing. No benefit from replacing it, or fix for our suspend
failure, was established. The audio host/DSP stack differs; the codec is shared.

Their Ubuntu installer provides pinned offline packages, dependency closure and
boot recovery machinery. Its release remains a physical-install candidate; the
repository acceptance template is not a completed recovery dossier. It targets
X91L and erases its selected internal eMMC. It is not a Debian/X90F upgrade path.

Their Validator adds a guided UI, microphone-signal analysis, charging/storage
sessions, package inventories, evidence expiry and multi-report acceptance.
Our generic pytest framework binds human evidence to exact machine/boot/kernel,
keeps intrusive tests opt-in and has distinct X90F hardware contracts. Useful
checks and reporting techniques can be exchanged without replacing the framework.
Reported project hardware observations were not independently reproduced here.

Their GNOME release refs contain real custom patches despite mirror descriptions:
Mutter orientation policy/sensor lifecycle, and a Settings output peak-meter
change avoiding inhibition of audio auto-suspend. These are narrow review
candidates, not justification for replacing Debian's entire GNOME stack.

Sources: [their UCM](https://github.com/Yoga-Book/Yoga-Book-ALSA-UCM-Config/blob/3a1a2a2c5c138fd331bd6bed4820bbbfdfdd802b/ucm2/cht-yogabook/HiFi.conf),
[SOF slot fix](https://github.com/Yoga-Book/Yoga-Book-Sound-Open-Firmware/commit/22f080bc3b359bacd0bc6a13e689d98706d44a20),
[installer release](https://github.com/Yoga-Book/Ubuntu-Autoinstall/releases/tag/ubuntu-26.04.1-yb1-x91l-2026.09.02),
[Validator](https://github.com/Yoga-Book/Yoga-Book-Validator/tree/bdf681dfc96d397dff8f2335a8521024864ad176),
[Settings patch](https://github.com/Yoga-Book/GNOME-Settings/commit/4d0790d73da9145c1631ca7d0ce3c97a266a764e).

## Contribution route and limits

Offer focused component changes, starting with input correctness and robust
sensor identification; discuss thermal readout independently of cooling policy.
Treat camera processing as an architecture discussion. Preserve the existing
jekhor submission series until a different review target is agreed; no wholesale
organization rebase is justified by this comparison.

Package identity is not lineage: both projects call unrelated implementations
`yogabook-camera`. UCM and other packages also share ownership paths. Do not mix
repositories or install their model-scoped packages onto the working device.

No source/ref/package/hardware changes, builds or hardware tests were made during
the comparison. No cross-project cherry-pick/rebase was attempted. Documentation
was saved afterwards at the user's request. Existing device evidence remains
bound to its recorded revision: the exact 6.17 submission built but was not
booted; runtime camera evidence comes from our LTS deployments. A submission
branch name does not establish upstream acceptance or passing physical gates.

## Subsequent targeted repairs (2026-10-28)

Kept the existing 6.18.58 hardware stack. Repaired LocalSearch's legacy SQLite
view and enabled Landlock in `#yb8.0`; real new-content search then passed.
The user accepted that kernel as the default. A controlled light test also
showed both ALS interfaces responding from 0 to 903.9 lux; no sensor workaround
was applied. The historical panel-brightness jump is still unconfirmed.
See the [runtime repair log](x90f-maintenance.md#current-kernel)
and [boot/rollback state](x90f-maintenance.md#boot-and-rollback).
