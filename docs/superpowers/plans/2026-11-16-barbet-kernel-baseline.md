# Barbet Kernel Baseline Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a reviewable Perpetua kernel baseline for Pixel 5a, preserving the existing Luma system until a gated hardware test is possible.

**Architecture:** Import only missing SM7250/Barbet support into the current Perpetua release. Keep new boot packing in Phonewave. Boot unchanged on Luma only after all required gates pass; proven regressions become separate fixes.

**Tech Stack:** ARM64 Linux C, DTS/YAML, Kconfig, LLVM, Python host tests, Android boot v2, UART and SSH.

**Spec:** `docs/superpowers/specs/2026-11-16-barbet-bringup-design.md`

## Global Constraints

- Kernel base: `0b3815eca32568b28eb308c59edb4ad2651bbbfa`, not Luma.
- Reference: `e52c61a6259a13101b45531d634b7e6e649305c4`; its clean delta/config and running-image match are in the external record.
- Keep Android slot A, firmware, calibration, bootloader, vbmeta, vendor_boot and dtbo untouched. No slot selection or automatic slot marking.
- New image packing belongs only in Phonewave. Do not add an in-tree v2 adapter or change `devices/check`.
- Gate 1 is blocked unless a conforming boot-chain registration mechanism is established; report this honestly and do not deploy around it.
- Private workspace `W=/home/bronson/luma-barbet-work/perpetua-barbet`; kernel object directory `O=$W/build-sm7250`.
- Use an isolated kernel worktree; no edits/builds in reference trees.
- One logical subsystem/component change per commit. Preserve authorship, use subsystem subjects and `cherry-pick -x` for upstream commits.
- No vendor stacks, board checks in drivers, textual C includes, out-of-tree modules or hardware-enabling module parameters.
- Existing behavioral fixes are excluded until a Perpetua test demonstrates the problem. Mandatory new hardware registrations must be distinguishable from behavioral changes.
- Record commands, results, image hashes, failures and recovery after every checkpoint. Raw logs/identities/blobs remain outside Git.

## Review Focus

- Wrong/transformed stock FDT: prove capture provenance and reversible transport correction; do not call the running Luma tree stock.
- Missing shared-driver prerequisites: identify them without silently importing the reference's behavioral changes.
- New SoC driver API drift: compile each directory with `W=1` and distinguish pre-existing warnings.
- False success from partial gates: retain failing checker/handoff output and prohibit deployment until all prerequisites pass.
- Wrong boot layout or output replacement: packing tests must reject malformed/oversize inputs and preserve existing outputs.

---

### Task 1: Close the survey and establish the implementation checkpoint

**Files:** external `$W/record.md`, `$W/reference-files.json`, `$W/stock-capture-provenance.json`; design and this plan.

**Interfaces:** consumes verified backups, reference delta and historical stock captures; produces a per-file device/present/shared/vendor/glue classification, fixture paths, exact BASE/worktree/O paths and a closed survey or explicit blocker.

- [ ] Validate the archived stock capture in `review-20261021/private-evidence/`: check magic, declared/actual length, dtc parsing and root identity; prove whether correcting PTY CRLF gives the retained corrected blob byte-for-byte.
- [ ] Record the corrected capture hash, original transport hash, captured Android command line, firmware provenance and what remains unverified. Never overwrite either original.
- [ ] Classify every path of `git diff --name-only 497791a0a96b866d90add64041c47acdff49cae6 e52c61a6259a13101b45531d634b7e6e649305c4`. Mark unrelated OnePlus files excluded and existing behavior deferred.
- [ ] Inspect posted SM7250 v10 series (`20261015051926.13752-1-sreeshankar0910@gmail.com`) before reference fallback; record exact source and authors for each imported change.
- [ ] Establish an isolated worktree using using-git-worktrees. Preserve the design/plan; record source and documentation commit boundaries so submission can exclude process documents.
- [ ] Commit the approved design and plan as a small documentation checkpoint only after the read-only survey is closed. Record unresolved submission-only provenance gaps separately.

### Task 2: Import missing foundational SoC drivers

**Files:** `drivers/clk/qcom/gcc-sm7250.c`, `drivers/pinctrl/qcom/pinctrl-sm7250.c`, `drivers/interconnect/qcom/sm7250.c`, associated binding headers/schemas and their Kconfig/Makefile entries.

**Interfaces:** produces `CONFIG_SM_GCC_7250`, `CONFIG_PINCTRL_SM7250`, `CONFIG_INTERCONNECT_QCOM_SM7250` and their compatible bindings. No QRTR/UFS/GPU/audio fixes.

- [ ] Configure host scratch O from arm64 defconfig with LLVM. Run explicit missing-object build targets to establish the pre-import failure.
- [ ] Import each foundational driver separately, preferring posted/upstream changes, adapting only APIs and minimal build/binding integration. Do not copy an entire combined reference commit.
- [ ] For each driver, build its object and directory with `ARCH=arm64 LLVM=1 O="$O" W=1`; run its binding checks where tools permit and `scripts/checkpatch.pl --strict` on its diff. Record missing validation tools as blockers, not passes.
- [ ] Commit each independent driver with its own binding/build integration and provenance. Update the external classification/evidence after each commit.

### Task 3: Describe the initial SM7250 and Barbet hardware

**Files:** `arch/arm64/boot/dts/qcom/sm7250.dtsi`, `sm7250-google-barbet.dts`, DT Makefile, relevant Qualcomm arm binding; narrowly scoped new SoC compatible registrations if allowed by the import classification.

**Interfaces:** produces `qcom/sm7250-google-barbet.dtb` and a list of omitted nodes/prerequisites. New hardware data must not drag in shared behavioral fixes.

- [ ] Establish the missing DTB target failure, then port the minimal SoC description onto Perpetua's existing shared DTS files.
- [ ] Add board wiring, memory reservations, regulators, UART, storage and USB supported by the available primitives. Omit unsupported feature nodes and record their missing dependencies.
- [ ] Add board root binding and build registration; compile the DTB and run applicable schema checks. Compare critical voltage/GPIO/reservation properties against the captured stock tree and reference source.
- [ ] Commit SoC additions apart from the board DTS, following contribution ordering; do not copy the reference's OnePlus board.
- [ ] If shared behavioral changes appear necessary to boot, stop at the policy boundary and report them instead of importing them.

### Task 4: Add missing board components without copying existing implementations

**Files:** missing touch/power/haptics drivers and their bindings/Kconfig/Makefile entries; board nodes as independent features.

**Interfaces:** enables only components actually absent from Perpetua; maintains a named deferred list for panel, GPU, UFS, remoteproc, QRTR and existing sound-driver changes.

- [ ] For each required missing component, record its failing build/binding target before importing.
- [ ] Port one component at a time with standalone bindings and existing kernel API/style conventions; no unrequested interactive hardware tests.
- [ ] Compile each affected directory with `W=1`, run binding/checkpatch checks and commit separately.
- [ ] Defer optional haptics and unrelated camera/video support if not needed for initial userspace. Do not add extra drivers just to match the reference's file count.

### Task 5: Integrate layered configuration and handoff; run host gates

**Files:** `arch/arm64/configs/sm7250.config`, `devices/barbet/device`, `devices/barbet/kernel.config`, board `/chosen` and `fdt-handoff`.

**Interfaces:** `devices/config barbet "$O"` produces a minimal layered config; builtin DTB names `qcom/sm7250-google-barbet`; current Luma ramdisk is retained.

- [ ] Establish unknown-device/config failure before registration.
- [ ] Set the board's own bootargs and handoff identity using captured device-specific bootloader words. Retain initrd and required board addresses.
- [ ] Add only the boot-critical SoC and device config values missing from existing layers. Do not claim a supported boot adapter when none fits.
- [ ] Run `ARCH=arm64 LLVM=1 devices/config barbet "$O"`; build `Image modules dtbs`; save complete logs. Resolve compile errors without speculative behavioral fixes.
- [ ] Run `devices/check barbet "$O" BASE`; retain all failures, including the known adapter contract. No passing claim if it exits nonzero.
- [ ] Run `devices/handoff/test` with the board and its captured tree/working boot input, expected exit 0; cross-device fixtures in both directions must exit 2. Save kernel physical range used and all outputs.
- [ ] Run checkpatch on each commit; rebuild/check existing devices affected by shared changes. Commit configuration and handoff in their logical places.

### Task 6: Prepare verified boot packing in Phonewave

**Files:** Phonewave `devices/barbet/boot/` packer and host tests; no kernel-tree packing code.

**Interfaces:** consume explicit working boot_b template, Perpetua Image and retained Luma ramdisk; emit a new verified 100663296-byte v2 image plus evidence. Never flash or replace outputs.

- [ ] Before implementation, write failing host cases for incompatible header, truncated inputs, oversize kernel/ramdisk, existing output preservation, payload roundtrip and absence of phone-access commands.
- [ ] Fix the exact CLI in Phonewave's device README, then implement minimal template-based packing: page 4096, preserved addresses, DTB, AVB algorithm NONE and rollback index 1783209600.
- [ ] Resolve ARM64 text_offset/placement explicitly and test it against the retained working image; do not silently alter unrelated Image bytes.
- [ ] Verify unpacked kernel/ramdisk/DTB, footer, length and hashes with real tools. Commit the packer/tests separately in Phonewave.
- [ ] Keep the kernel registration blocker open unless an already-supported conforming mechanism or maintainer clarification resolves it.

### Task 7: Conditional hardware baseline, fixes and Denial handoff

**Files:** external record/logs; independently scoped proven fixes; later Phonewave/Denial plan.

**Interfaces:** requires ALL Gate-1 checks, verified recovery and sufficient battery. Produces unchanged-import boot evidence or a recovered, diagnosed failure; then a feature matrix and second-stage installation plan.

- [ ] If the adapter or any other required gate remains blocked, stop before deployment and report exact failing commands. Do not bypass a gate.
- [ ] Otherwise install matching modules alongside the reference release, announce the single boot_b write and recovery image, verify complete readback, then boot under UART capture.
- [ ] Record release, handoff marker and power supplies. Check charging before extended tests; return to reference to charge if needed.
- [ ] Reproduce each regression before selecting an upstream/reference/new fix; build/check/deploy/re-test one logical fix at a time and commit verified results.
- [ ] Obtain user confirmation for visible display/touch behavior and permission for interactive tests. Keep thermal/sensors and unqualified reference functions explicitly tracked.
- [ ] After the kernel baseline and USB-SSH rescue are established, write the second-stage Phonewave/Denial plan covering shared early userspace in boot_b, tail layout, private firmware, service ordering, audio routing and userdata installation. Do not erase Luma as part of this kernel-baseline plan.

## Execution and self-review

User approved the design and explicitly requested building it. Execute inline now;
do not dispatch implementation agents. The earlier builder review is complete.
All five review-focus items have explicit checks in Tasks 1, 3, 2, 5/7 and 6
respectively. Kernel/Phonewave histories and development/submission evidence
stay separate. Known gate dependencies are intentionally not treated as passes.
