# Provisioning checkpoint — 2026-09-07

This checkpoint fixes target-chroot command resolution, propagates provisioning failures, retains installation logs in `@log`, restores installed-service settings, and protects shared ESP ownership. Chromium's software-GL wrapper must exist before user finalization: the browser desktop entry refers to it, and `xdg-settings` rejects a missing executable.

## Validation

- ISO `./test/unit` and `git diff --check`: pass.
- `test/provisioning-chroot` with the rebuilt desktop package, real bind mounts, chroot, and disposable btrfs `@log`: success and failure scenarios pass. Its setup commands are synthetic; it is not a full desktop installation test.
- The original image failed user finalization on an M3. A disposable-overlay rehearsal reproduced the failure with the old ordering and passed with the corrected ordering. The incomplete installation was then recovered successfully. The ESP checksum and partition table were unchanged by that recovery.
- A full `--usb --rootfs` build completed at `2026-09-07T11:59:20Z`, recreating the root filesystem, live/encrypted/plain initrds, and GRUB loaders. The delivered artifact is not the intermediate surgical payload patch.
- Kernel input: `7.2.2-omarchy-wip72+`, source ref `236788cd2602a24c703fe7bdaddaf73ef77d2027`, with matching module inputs. Desktop packages: `omarchy` and `omarchy-settings` `4.0.2-4`; video packages `avd-fw` `0.1-1` and `libva-v4l2_request-avd` `1.3-1`.
- Build checksums, packaged-source comparisons, initrd hooks and required modules, module vermagic, and final hardware database lookup: pass. The installer extracted directly from the raw payload matches source. Decompressing the delivered payload reproduces the verified raw-image hash.
- The portable tester drop was copied to the transfer volume, flushed, and every manifest entry verified. Older files were preserved.

The desktop companion changes avoid privileged permission changes for an already-correct Electron wrapper, refuse boot-to-ESP migration for shared/UUID-private layouts, and test the timezone notification's argument handling. Their focused tests pass. Earlier full desktop QA had five environment-dependent shell-test failures; this checkpoint does not claim that the entire desktop suite is green.

The builder emitted an intermediate hwdb hook error and kernel-image autodiscovery/optional-firmware warnings. Final hardware database and required initrd checks pass; those checks do not prove hardware firmware availability.

## Artifact identity

Tester folder: `omarchy-mac-apple-silicon-preview-20260907-wrapper-order`.

- Raw payload SHA-256: `ebc207bb1f2db40fce767e5d478c0a7483d0c1c7050577e467359676f0c8df32`.
- Compressed payload SHA-256: `abe636dae010c5dc1e77bbb8c89ea6b5823303763f3dc1a200555b3a40026015`.
- Packaged installer SHA-256: `3d499821312462593594f1296fcd3f38447cbe39f49ed66af27d6e03b4280e9d`.

The build predates this checkpoint commit and records its source as dirty. These hashes identify the tested artifact; no byte-for-byte reproducibility claim is made. Detailed local logs remain under the ignored `release/install-audit-20260907/` directory and are not required to run the committed regression tests.

## Remaining hardware validation

At commit time, a fresh installation of the rebuilt image is in progress. Before reboot, inspect the complete provisioning log, completion markers, and root-specific boot configuration. After reboot, validate LUKS unlock, first graphical login, absence of unattended sudo, notification actions, networking authorization, clock synchronization, and installer-slice consumption/growth. Recovery of the previous installation is not proof that this new image completes an end-to-end installation. This is a work-in-progress checkpoint, not a release sign-off.
