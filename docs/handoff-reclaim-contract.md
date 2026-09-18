# Handoff and reclaim contract

Three stages install Omarchy on an Apple Silicon internal disk without a USB stick: **place** writes a temporary installer slice from macOS, **install** copies the system into the space in front of it, **reclaim** deletes the slice on first boot and grows the root into it. Handoff is the state one stage leaves for the next.

MUST, MUST NOT, and MAY are used as in RFC 2119. Rules are numbered (S-, H-, P-, I-, R-) so scripts and tests can cite them.

## Terms

- **Installer slice** — the GPT partition holding the live payload (btrfs, label `OMARCHYLIVE`), GPT name `omarchy-install`.
- **Hole** — unallocated GPT space in front of the installer slice.
- **Apple partitions** — APFS, iBoot, Recovery, and the ESP firmware trees `m1n1/`, `vendorfw/`, `asahi/`.
- **New root** — the Omarchy partition created by install.

## Handoff

Exactly two artifacts carry state between stages. Nothing else is a channel.

- H1. GPT name `omarchy-install` on the installer slice.
- H2. `omarchy-mac-consume-installer.service` enabled in the new root (`multi-user.target`).

## Shared rules

- S1. Apple partitions MUST NOT be shrunk, erased, reformatted, or deleted. `m1n1/`, `vendorfw/`, and `asahi/` MUST be hash-identical after every stage.
- S2. Unclear identity MUST refuse. No stage deletes by disk position, size, or elimination.
- S3. The unlock password MUST NOT be written anywhere: macOS, ESP, logs, installer slice.

## Stage 1 — Place (macOS)

Preconditions: Asahi UEFI-only provision done (ESP has `m1n1/`). One GPT hole at least payload plus reserve in size.

Rules:

- P1. The installer slice MUST sit at the tail of the hole. The space in front MUST stay unallocated.
- P2. Only the payload image MUST be written to the slice, at the disk's native block size. The full USB image MUST NOT be written.
- P3. GPT name `omarchy-install` MUST be attempted. An empty name is acceptable; any other name is not.
- P4. The placer MAY replace `EFI/BOOT/BOOTAA64.EFI` and add kernels and initrds to the ESP. S1 holds.
- P5. The placer MUST dry-run unless `--confirm` is given.

Postconditions: `[hole] [installer slice] [Recovery]`. Payload on the slice. Live GRUB on the ESP.

## Stage 2 — Live install

Preconditions: booted from the installer slice on NVMe.

Rules:

- I1. On boot, the slice MUST be named `omarchy-install` if macOS left it empty (H1).
- I2. Install MUST target the hole only. The running slice MUST NOT be a target.
- I3. The new root MUST be LUKS with label `OMARCHYROOT`, unlocked by the user password. Goal vs today: the TUI still offers unencrypted with encryption as the default.
- I4. ESP mode: piggyback (`custom.cfg` only; `BOOTAA64.EFI` hash unchanged) when another loader owns `BOOTAA64.EFI`; own when none exists or the placer wrote it.
- I5. The reclaim service MUST be enabled in the new root (H2). Install MUST NOT delete the slice or grow.

Postconditions: encrypted root in the hole. Bootloader points at it. Installer slice present and named. Service enabled.

## Stage 3 — Reclaim (first boot)

Preconditions: root on NVMe. `/omarchy-mac-overlay-write` absent.

Rules:

- R1. The target is the partition on the root's disk with GPT name `omarchy-install`. It MUST NOT be the running root, mounted, or an Apple partition type.
- R2. Fallback: when no partition has that name, a partition with filesystem label `OMARCHYLIVE` that is not the root MAY be named `omarchy-install` first. R1 still gates the delete. Goal vs today: the goal is exactly one candidate, else refuse; the code takes the first match.
- R3. The root MUST grow only into the hole the delete left, then `cryptsetup resize` if LUKS, then `btrfs filesystem resize max`. It MUST NOT grow to end of disk.
- R4. Apple partition entries MUST be snapshotted before the delete and MUST be identical after the delete and after the grow. Any difference fails the run.

Postconditions: no `omarchy-install` on the disk. Root grown. Service disabled. A re-run is a no-op.
