# Pixel 5a: Perpetua first, then Denial Mobile

Status: design approved by the user; implementation authorized.

Submission contract: `.github/CONTRIBUTING.md`, read in full following the
user's request. The kernel submission targets `perpetua/v7.3`; userspace,
Phonewave packages and boot-image packing are separate submissions once the
kernel is in. Do not publish either PR without a separate request.
The user explicitly chose this submission boundary over adding an in-tree
generic v2 adapter. Do not add such an adapter or change the checker contract
as part of Barbet support.

## Intent

Bring Google Pixel 5a (`barbet`, Qualcomm SM7250) up on linux-perpetua, using
the recent Luma port as evidence rather than as a kernel base. Then supply
the Phonewave integration needed to run Denial Mobile. The user wants small,
logical commits and an updated record so work can resume without repeating
hardware investigations.

The user permits replacing the Linux userspace, but Android slot A must
remain untouched. `userdata` is shared between slots; preserving slot A
does not preserve Android user data.

## Approaches

1. **Kernel first on existing Luma, then Denial — selected by the user.**
   Preserve a known userspace while checking kernel behavior. Once Perpetua
   boots and its hardware baseline is understood, adapt the userspace to
   Phonewave and install Denial. This adds a checkpoint but separates kernel
   failures from userspace failures.
2. **Change kernel and userspace together — rejected.** Shorter nominal
   installation path, but failures in storage, firmware, remoteproc startup,
   routing or graphics become difficult to attribute.
3. **Merge Luma's kernel wholesale — excluded by this tree's procedure.**
   Luma's 7.2.15 source is not Perpetua's base. Its unrelated OnePlus code
   and shared behavioral fixes must not enter merely because they exist.

This is a project with two sequential deliverables. The first implementation
plan covers the kernel baseline. Phonewave/Denial installation gets its own
plan after that baseline, using the interfaces and constraints below.

## Established evidence

- Starting Perpetua commit:
  `0b3815eca32568b28eb308c59edb4ad2651bbbfa`, branch `perpetua/v7.3`.
- Running Luma reference:
  `e52c61a6259a13101b45531d634b7e6e649305c4` on Linux 7.2.15.
  Its complete clean delta and matching build configuration are retained.
- A captured UART boot reached Luma's login prompt on
  `7.2.15-ge52c61a6259a`. SSH and sudo work. Wi-Fi associated.
- Fresh full backups of both slots' boot, vendor_boot, dtbo, vbmeta and
  vbmeta_system partitions passed length and second-read SHA-256 checks.
- The unlocked bootloader was observed on slot B. Both slots were marked
  successful and bootable. Battery was reported full; current and charging
  rate were not available.
- Working Linux boot_b is Android header v2, 4096-byte pages, 100663296
  bytes, with an LZ4-frame kernel, ramdisk and DTB in the boot image.
  Its AVB footer is algorithm NONE, rollback index 1783209600.
- Android boot_a is header v3 and signed. It is not the Linux packing
  template. vendor_boot_b is not assumed stock.
- The working Luma packer sets ARM64 Image text_offset to 0x80000.
  This is a boot-placement issue to understand and verify, not permission
  to copy an unexplained header hack.
- Luma boot logs already contain display PLL, early firmware lookup and
  deferred audio probe warnings. Compare runtime results, not warning count.

Private evidence and operational identities live in
`/home/bronson/luma-barbet-work/perpetua-barbet/record.md` and its adjacent
artifacts, not in either repository. That record is authoritative for
current device state, backup hashes, exact recovery commands and test logs.

## Goals and acceptance

### Deliverable 1: recoverable Perpetua kernel baseline

- Register `devices/barbet/` and a dedicated built-in board tree, on this
  tree's configuration and handoff conventions.
- Build `Image modules dtbs` and pass `devices/check`, applicable binding
  checks, handoff tests and checkpatch.
- Produce a verified boot image without touching protected partitions.
- Boot the unchanged import into existing Luma userspace, record `uname -r`
  and the empty `/chosen/linux,fdt-handoff` marker.
- Establish charging state first. Then compare display, touch, storage,
  USB, Wi-Fi, GPU, keys, audio and suspend against the reference.
- Fix only demonstrated regressions, one independently tested change at a
  time. A partial baseline is explicitly partial, not device qualification.

### Deliverable 2: Denial Mobile through Phonewave

- Supply the device facts, boot packaging, firmware inventory and necessary
  service/routing configuration under Phonewave's existing conventions.
- Boot an Arch Linux ARM/Denial userspace with reachable USB SSH, usable
  display/touch, working graphics and documented hardware limitations.
- Confirm the visible shell with the user; a running process or screenshot
  is not enough to claim the screen works.
- Distinguish reference parity, initial Denial usability and full hardware
  qualification in both repositories' documentation.

## Non-goals

- No promise of working calls, mobile data, cameras, fingerprint, NFC,
  Bluetooth or headset support: these are unqualified on the Luma reference.
- Do not upgrade partial microphone capture into a claim of qualified audio.
- No vendor framework import, kernel-version replacement, generic installer
  rewrite, unrelated OnePlus port or new charging-performance project.
- No firmware redistribution, release publishing, pushing or PR creation
  without a separate request.
- Do not implement the unimplemented hazards proposal as part of this port.

## Safety and recovery

- Never write slot A, partition tables, bootloader, vbmeta, vendor_boot,
  dtbo, firmware, calibration or modem identity partitions.
- Never change the active slot or mark slots successful automatically.
  If the bootloader falls back to A, stop and ask; do not switch it back.
- Kernel trials write only `boot_b`, one image at a time, after naming
  partition, slot, image, reason and verified recovery image.
- Keep the known Luma kernel and userspace usable until Perpetua's charging
  is established. Start trials only with enough battery for test/recovery.
- Before every trial, verify device identity, slot, image size/hash and
  UART access. Verify the complete partition readback before rebooting.
- Preferred deployment is through the running system over SSH, following
  the established Luma safety ledger. A recovery flash of the backed-up
  boot_b through fastboot is the fallback, with the same write announcement.
- Preserve the backed-up working Linux boot image byte-for-byte. Physical
  bootloader access is demonstrated; record the exact key sequence before
  deployment. Do not describe a historical backup as a current readback.
- Later userdata replacement requires a tested Perpetua rescue with USB SSH,
  a recovery record and a separate installation checkpoint. Do not overwrite
  Luma while still diagnosing the first kernel boot.
- No pairing, vibration, playback or other visible/interactive hardware
  events unless the user asks for that test. The user validates the screen.

## Kernel integration

Follow `.claude/skills/add-device/SKILL.md` and `devices/README`: survey,
import what is missing, boot unchanged, then fix proven failures.

The survey must close before importing: complete file classification,
pin reference config/modules/delta, and capture or reconstruct a correctly
labelled bootloader-tree fixture. The running Linux FDT is useful for
memory and chosen fixups but is not a stock-tree capture. For submission,
CONTRIBUTING specifically requires a stock Android `/sys/firmware/fdt`
capture, its firmware version and attachment to the PR. Reconstruction
alone is not submission evidence. Find a genuine retained stock capture;
if unavailable, arrange a separately approved stock capture without
automatically changing slots or overwriting protected partitions.

For each missing SoC primitive or board component, prefer an upstream
commit, then a posted patch, then an adapted reference implementation.
Record authorship and provenance. Separate minimal new hardware support
and its build/binding registration from changes to existing behavior.
Already-present driver implementations remain this tree's implementations.
If initial boot requires a shared fix disallowed by the import procedure,
stop at the gate and ask rather than silently import it.

Expected boundaries:

- SM7250 bindings, clocks, pinctrl, interconnect and basic SoC description:
  small subsystem-focused commits, excluding unrelated OnePlus hardware.
- `arch/arm64/boot/dts/qcom/sm7250-google-barbet.dts`: wiring, rails,
  reservations and devices supported by the imported baseline.
- Missing component drivers such as ST FTS SPI touch: standalone driver,
  binding and build integration, respecting current APIs.
- Existing panel, UFS, GPU, remoteproc, QRTR and sound changes: examine as
  evidence; do not transplant their reference versions.
- `arch/arm64/configs/sm7250.config` and
  `devices/barbet/{device,kernel.config}`: minimal layered fragments, not
  Luma's complete configuration.
- `/chosen/fdt-handoff`: stable device identity from bootloader-generated
  words, retained initramfs and required address handoffs. Test against
  stock and working boot inputs and all available other-device fixtures.

For an enabled DTS node, supply its required binding and driver support.
Leave unsupported nodes out and name them in the external record.

## Boot packaging contract

Per CONTRIBUTING, new Barbet boot-image packing belongs in Phonewave,
not in the kernel submission. The adapter consumes explicit template, Perpetua Image,
ramdisk and output inputs; it never accesses a phone or flashes anything.
Its exact CLI and name are builder decisions fixed in the implementation
plan after the packing survey.

**Known Gate-1 tooling blocker:** existing
`devices/check` requires the device's `boot` field to name an executable
under `devices/boot/`. `devices/README` and the add-device procedure also
describe adding generic chain adapters there, while CONTRIBUTING places
boot-image packing in its own repository. No existing adapter supports
the verified v2 layout. Do not name an incompatible adapter, weaken the
checker or add an unexplained shim just to pass. The user's decision is
to follow CONTRIBUTING without extra kernel tooling changes. Keep new
packing in Phonewave and record the precise registration/check failure.
Source inspection, provenance work and host builds can proceed; do not claim
Gate 1 passes or deploy a new kernel while the required check is blocked.
Resolve through an already-supported upstream mechanism or maintainer
clarification, not a new external-adapter contract invented for this port.
If no conforming mechanism exists, stop and report this dependency.

Preserve the working template's header and AVB properties where applicable.
Preserve an appropriate bootloader-facing DTB while Perpetua uses its
built-in tree; do not assume the bootloader accepts an arbitrary DTB.
Understand kernel placement and prove that both decompression and handoff
use safe ranges. Preserve the original Luma ramdisk for initial trials.
The final image must fit 100663296 bytes and pass unpack/payload comparison
and AVB verification. Reject incompatible templates, oversize inputs,
existing outputs and failed verification without publishing a partial image.

When Phonewave replaces Luma, pack its shared early userspace into Barbet's
boot ramdisk. Do not invent an init_boot partition. Reuse common early
userspace, boot-tail and package mechanisms only after checking their
header-v2 layout assumptions.

## Phonewave boundary

Work in `/home/bronson/Projects/phonewave`, with a separate small-commit
history. Device-specific facts and files belong in `devices/barbet/`;
reusable services stay in existing service directories.

Audit Luma's startup dependencies instead of copying its overlay wholesale:
firmware locations, read-only rmtfs, tqftpserv, modem/WLFW-before-ADSP ordering,
audio routing/UCM, permissions and suspend-related USB handling. Identify
remoteprocs by hardware identity, not assumed enumeration numbers.

Do not carry Luma-specific OSTree/GNOME integration or slot-confirmation
services. Use Phonewave's generic Arch/Denial image and private firmware
vault conventions. Firmware payloads and SSH keys stay outside Git.

## Verification and checkpoints

- Finish and record the survey; then preserve the design and plan as their
  own documentation commit.
- Commit by subsystem/component, with the relevant build or host-test
  result recorded. Do not claim a kernel/device gate from a partial build.
- Gate 1: configuration, full build, devices/check with pre-import BASE,
  handoff positive and cross-device negative cases, checkpatch, boot-image
  roundtrip and hashes. Rebuild affected existing devices for shared changes.
- Gate 2: unchanged import boots Luma, reports the correct release and handoff
  marker, with boot and power-supply evidence saved.
- Step 3: reproduce, diagnose, minimally fix, rebuild/check/deploy and
  re-test each regression. Charging takes priority; no speculative fix bundle.
- Stop before the Denial installation until the kernel baseline, rescue and
  remaining feature limits are documented. The second plan must explicitly
  cover header-v2 early userspace/tail integration and userdata installation.
- Prepare submission evidence in the CONTRIBUTING format: device/SoC/stock
  firmware, attached stock FDT, exact configuration/full-build/check output,
  handoff results, and display, touch, storage, USB, Wi-Fi, Bluetooth,
  audio, charging, battery, sensors, thermal and suspend marked works,
  partly works (with the gap) or does not work. Keep not-yet-tested items
  explicitly untested during development; do not invent a result for the
  final feature list.
- Order commits as the contribution guide specifies: upstream backports
  (`cherry-pick -x`), SoC additions separate from boards, bindings/drivers,
  board tree then per-feature changes, configuration fragments. Use
  subsystem subject conventions; new Signed-off-by lines are not required.
  Preserve original attribution on imported work.

## Builder decisions

The implementer selects upstream commit sources, minimal SoC bring-up scope,
Phonewave packer CLI/name, host-test framework, build output paths and exact commit
boundaries after inspecting the dependencies. Those choices cannot relax
device safety, contribution boundaries or gates. A broader driver rewrite, vendor-stack replacement
or need to change protected partitions returns to the user for approval.
