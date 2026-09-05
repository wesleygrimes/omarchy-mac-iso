Omarchy on Apple Silicon — M3 tester drop
========================================

This is an early Omarchy installer for Apple Silicon. It is not an ISO.
Do not dd omarchy-mac-usb.img onto the internal SSD.

Proven so far: 13" M3 Air (j613) with a usable panel (full resolution +
notch). That is the display controller, not the 3D GPU. There is no AGX
yet — the desktop is CPU-composited.

We think the installer itself will run on other M3 laptops (15" Air,
Pro, Max) and probably the M3 iMac. Display on those may still be the
old simple framebuffer. If you have a 15" Air, Pro, Max, or iMac, please
install if you are willing, then run the dump command at the end and
send the tarball back.

Not this drop: M3 Ultra (Mac Studio).

You will wipe the Linux slice only. macOS, Recovery, and the Asahi EFI
(m1n1) stay. If you are not willing to do that, stop here.


0. What should be in this folder
--------------------------------
  payload.img.zst
  vmlinuz-linux-asahi
  initramfs-omarchy-usb.img
  initramfs-linux-asahi.img
  BOOTAA64.EFI
  grub-nvme-installer.cfg
  place-nvme-installer.sh
  README.txt (this file)
  src/                   kernel config, DCP overlay source (GPLv2)

Optional: initramfs-linux-asahi-plain.img (only if you install without
encryption).

On the Mac: brew install zstd   (the placer decompresses payload.img.zst)


1. Asahi UEFI-only (macOS), if you do not already have it
---------------------------------------------------------
A Mac that has never run the Asahi installer cannot boot this. iBoot
will not load our GRUB until m1n1 is on the internal EFI.

From Terminal in macOS (M3 needs the expert/dev installer, firmware
14.8.3):

  export EXPERT=true
  curl -L https://alx.sh/dev | sh

Choose **UEFI environment only**. Do not install Fedora/Alarm as the OS.
Let it put firmware on the ESP (m1n1, vendorfw, asahi). Shrink APFS in
Disk Utility (or the Asahi step) so you have a **hole of free space**
for Linux. Aim for roughly twice the payload plus slack — on the order
of 30 GB or more, more if you want a large Omarchy root.

If Linux is already in that hole from an earlier experiment, you can
turn **only** that Linux/LUKS slice back into free space:

  diskutil list disk0
  sudo diskutil eraseVolume free none disk0sN

N is the Linux identifier you just wrote down. Never eraseDisk. Never
touch APFS, Recovery, iBoot, or the ~500 MB EFI that already has m1n1/.


2. Place the Omarchy installer (still macOS)
--------------------------------------------
Put the files from this folder in one directory, cd there, dry-run
first:

  sudo ./place-nvme-installer.sh \
    --payload ./payload.img.zst \
    --esp-files .

You should see a plan like: hole in front, omarchy-install at the tail,
then Recovery. If that looks right:

  sudo ./place-nvme-installer.sh \
    --payload ./payload.img.zst \
    --esp-files . \
    --confirm

The script will not mklabel the disk. On the internal SSD, gpt add is
EPERM from macOS — it uses diskutil instead. dd is only onto the new
slice (never rdisk0 / the whole disk). It copies live GRUB next to
m1n1/; m1n1, vendorfw, and asahi must stay byte-identical.


3. Boot and install
-------------------
Shut down, then power on (a warm reboot often misses Type-C). You do
not need a USB stick if place succeeded — U-Boot should load NVMe
EFI/BOOT/BOOTAA64.EFI.

If the machine already had Linux on NVMe, interrupt U-Boot so NVMe GRUB
does not win. Do not saveenv.

In the live TUI: **Install into free space** (the hole in front of the
installer slice). Not replace-existing, not wipe USB.

Encrypt is recommended (same password as the desktop user). Confirm the
review table.

M3 Type-C: if a USB device does not show up, unplug and plug it back in
once the desktop or live session is up.

First boot of the new root should delete only the temporary
omarchy-install slice and grow into that space. If `lsblk` shows the
LUKS partition larger than the btrfs inside it, run:

  sudo cryptsetup resize root
  sudo btrfs filesystem resize max /


4. What we want you to dump (especially M3 Pro / Max / 15" Air / iMac)
----------------------------------------------------------------------
After you can log in:

  sudo omarchy-mac-dump-board

It is read-only (it does not write boot.bin or firmware). It writes a
tarball under /tmp, named like:

  /tmp/omarchy-mac-board-j514s-YYYYMMDDThhmmssZ.tar.zst

Send that file back. It has board compatible/model, whether apple-drm
or simpledrm bound, USB/Type-C, the running device tree, and the DTB
slots from m1n1. That is what we use to add a display overlay for your
machine if the panel is still the simple framebuffer.

Quick sanity (optional to include in the same message):

  uname -r
  cat /sys/class/graphics/fb0/virtual_size
  ls /sys/class/drm
  lsblk -o NAME,SIZE,FSTYPE,LABEL,PARTLABEL


5. What “good” looks like
-------------------------
13" M3 Air: kernel 7.2.2-omarchy-wip72+, display ~2560x1664 with notch,
apple-drm, USB after a replug, macOS still there.

Other M3s: we want to know if the installer completed, what the
resolution is, and the dump tarball — even if graphics are ugly.
