# iMac 27" 5K (iMac15,1) — Fedora KDE + LibreWolf + True 5K — One USB Stick

> **This guide is written exclusively for the iMac 27" 5K Late 2014 (iMac15,1).**
> No macOS required at any point. One USB stick. Full 5120×2880 output in Linux.

---

## Table of Contents

1. [Your Machine — iMac15,1 Specs & Linux Compatibility](#1-your-machine--imac151-specs--linux-compatibility)
2. [The One-USB Plan — How This Works](#2-the-one-usb-plan--how-this-works)
3. [Why 5K Needs OpenCore](#3-why-5k-needs-opencore)
4. [What You Need](#4-what-you-need)
5. [Phase 1 — Write Fedora to the USB Stick](#5-phase-1--write-fedora-to-the-usb-stick)
6. [Phase 2 — Install Fedora to the iMac's Internal Drive](#6-phase-2--install-fedora-to-the-imacs-internal-drive)
7. [Phase 3 — First Boot, Fix Wi-Fi](#7-phase-3--first-boot-fix-wi-fi)
8. [Phase 3 — Post-Install Essentials](#8-phase-3--post-install-essentials)
9. [Phase 4 — Reuse the USB Stick for OpenCore](#9-phase-4--reuse-the-usb-stick-for-opencore)
10. [Phase 4 — Build OpenCore EFI in Fedora](#10-phase-4--build-opencore-efi-in-fedora)
11. [Phase 4 — Write OpenCore to the USB Stick](#11-phase-4--write-opencore-to-the-usb-stick)
12. [Phase 5 — Test Boot via OpenCore USB](#12-phase-5--test-boot-via-opencore-usb)
13. [Phase 5 — Move OpenCore to Internal EFI (Ditch the USB)](#13-phase-5--move-opencore-to-internal-efi-ditch-the-usb)
14. [Phase 6 — Enable True 5120×2880](#14-phase-6--enable-true-5120x2880)
15. [Phase 6 — Set 200% HiDPI Scaling in KDE](#15-phase-6--set-200-hidpi-scaling-in-kde)
16. [Install & Configure LibreWolf](#16-install--configure-librewolf)
17. [Set Your Wallpaper at 5K](#17-set-your-wallpaper-at-5k)
18. [Troubleshooting & Known Issues](#18-troubleshooting--known-issues)

---

## 1. Your Machine — iMac15,1 Specs & Linux Compatibility

| Component | Detail |
|-----------|--------|
| Display | 5120×2880 IPS, 218 ppi |
| CPU | Intel Core i5-4590 or i7-4790K (Haswell) |
| GPU | AMD Radeon R9 M290X (2 GB) or M295X (4 GB) |
| Wi-Fi | Broadcom BCM4360 — needs proprietary driver |
| RAM | 8–32 GB DDR3 1600 MHz (user-upgradeable) |
| Storage | 1 TB HDD or Fusion Drive / PCIe SSD |
| Ethernet | Gigabit — works out of box in Linux |

### Linux compatibility

| Feature | Status |
|---------|--------|
| CPU / RAM / storage | Works perfectly |
| AMD R9 M290X / M295X | Open AMDGPU driver |
| Ethernet / Audio / Bluetooth | Works out of box |
| Broadcom BCM4360 Wi-Fi | Needs `broadcom-wl` from RPM Fusion |
| 5K (5120×2880) via OpenCore + nomodeset | **Community-confirmed working** |
| 5K via amdgpu MST kernel params | Works on some GPU variants |
| Without OpenCore | Hard-capped at 3840×2160 |

> **Confirmed:** Community members on iMac15,1 have posted xrandr output showing `current 5120x2880` running Linux via OpenCore. The `nomodeset` path is the most reliably confirmed route. This entire guide is written for this machine only.

---

## 2. The One-USB Plan — How This Works

The USB stick gets used twice, then is no longer needed:

**Phase 1–2: USB = Fedora installer**
Write Fedora KDE to the USB. Boot the iMac from it. Install Fedora to the internal drive. After install, remove the USB — Fedora runs from the internal drive.

**Phase 3–4: USB = OpenCore bootloader**
From within Fedora (now on the internal drive), reformat the USB as FAT32 and write the OpenCore EFI to it. Boot via OpenCore on the USB to get 5K.

**Phase 5: Copy OpenCore to internal EFI — USB freed permanently**
Once 5K is confirmed, copy OpenCore from the USB to the iMac's internal EFI partition. The USB is no longer needed and can be used for anything else.

> End result: no USB needed for daily use. OpenCore lives on the internal EFI and loads automatically at every boot.

> **Minimum USB size: 8 GB.** Fedora needs ~2 GB but the drive gets reformatted between phases anyway.

---

## 3. Why 5K Needs OpenCore

### The firmware problem

The iMac15,1 drives its 5K panel using a dual DisplayPort 1.2 configuration — two streams stitched into 5120×2880. Apple's firmware detects any non-Apple UEFI bootloader (GRUB included) and immediately drops the display controller to a single stream, capping output at 3840×2160. This is intentional and affects Windows, Linux, and every third-party UEFI tool equally.

### How OpenCore bypasses it

OpenCore loads through a firmware trust path Apple left open for diagnostics tools. By using that trusted path, OpenCore initialises before the firmware degrades the display controller — preserving the 5K dual-DP stream. OpenCore then chain-loads Fedora's GRUB. The display stays at 5K for the entire Linux session. No macOS is needed — the entire OpenCore EFI is built from within Fedora.

### Two confirmed 5K paths once OpenCore is running

| Method | Status |
|--------|--------|
| `nomodeset` — VESA framebuffer | Most reliably confirmed |
| amdgpu MST kernel params | Works on many M290X/M295X variants |

Try amdgpu MST first — it gives full GPU acceleration. If it produces a split-tile display or black screen, switch to `nomodeset` which is confirmed to show a single clean 5K display (at the cost of hardware acceleration).

> **Read sections 10–14 in full before starting.** Building OpenCore manually from Linux is involved. A misconfigured EFI produces an unbootable system.

---

## 4. What You Need

- **One USB stick — 8 GB or larger.** Used twice, then permanently freed.
- **Wired keyboard.** You must hold `Option (⌥)` at boot multiple times. Bluetooth keyboards don't work reliably before the OS loads.
- **Ethernet cable or USB phone tether.** Broadcom Wi-Fi needs a manual driver after first boot. Android USB tethering and iPhone Personal Hotspot via USB both work automatically in Fedora.
- **Another device to read this guide.** You'll be rebooting the iMac multiple times. Keep this guide open on a phone or another computer.
- **Full backup of anything on the iMac.** This guide wipes the entire internal drive. No data can be recovered afterwards without a backup.

---

## 5. Phase 1 — Write Fedora to the USB Stick

> Do this on any computer that has internet access.

**1. Download Fedora KDE Plasma Desktop v42**

Go to `fedoraproject.org/kde/download` and grab the Live ISO for x86_64. Fedora 42 is the current stable release as of March 2026.

**2. Write to USB using Fedora Media Writer**

Download Fedora Media Writer from the same page. Open it → Select .iso file → select USB → Write.

> **Warning:** Everything on the USB stick is permanently erased.

**3. Alternatively — write with dd (Linux/macOS)**

```bash
sudo dd if=Fedora-KDE-Live*.iso of=/dev/sdX bs=4M status=progress
```

Replace `/dev/sdX` with your USB device. Use `lsblk` to identify it — be absolutely certain it's the USB and not your main drive.

**4. Verify checksum**

```bash
shasum -a 256 Fedora-KDE-Live*.iso
```

Compare to the SHA256 on the download page. Fedora Media Writer does this automatically after writing.

---

## 6. Phase 2 — Install Fedora to the iMac's Internal Drive

> USB stick plugged in — iMac boots from it.

**1. Plug in USB stick, restart iMac, hold Option (⌥)**

Hold `Option (⌥)` the instant the screen goes dark after restart. Keep holding until the firmware boot picker appears. Select "EFI Boot" — the orange USB icon. Press Enter.

**2. At GRUB, choose "Start Fedora-KDE-Live"**

Boots into a live Fedora desktop. Nothing is written to the internal drive yet.

> **Tip:** Black or garbled screen after GRUB? Press `E` at the GRUB menu. Add `nomodeset` after `quiet splash` on the linux line. Press `Ctrl+X` to boot. This is normal for the AMD GPU on first boot.

**3. Click "Install to Hard Drive" on the desktop**

The Anaconda installer opens. Select language and keyboard layout.

**4. Partitioning — choose Automatic**

Automatic wipes the internal drive entirely and creates the correct partition layout: EFI partition (~600 MB FAT32), root partition (btrfs by default in Fedora 42), and swap. You will add OpenCore to the EFI partition in phase 4.

> **Note on filesystem:** Fedora 42 defaults to **btrfs** for the root partition. Remember this — you'll need the matching filesystem driver for OpenCore in section 10. To force ext4 instead, choose Custom partitioning and create the root partition manually. Either works — just remember which you chose.

> **Danger:** Automatic mode wipes everything on the internal drive. Confirm your backup is complete before clicking Done.

**5. Create user account and install**

Set username and password. Check "Make this user administrator". Click Begin Installation. Takes 5–15 minutes.

**6. When done — click Quit, not Restart Now**

Click **Quit** to return to the live desktop. Do not reboot yet. The USB stick is still running Fedora live — you need it for this next step.

**7. Reboot — remove USB stick when the screen goes dark**

Now restart normally. Pull the USB stick out the moment the screen goes dark. The iMac boots Fedora from the internal drive via GRUB. Log in — the display will be at 3840×2160 for now. That is expected.

---

## 7. Phase 3 — First Boot, Fix Wi-Fi

> Fedora running from internal drive — USB stick not needed yet.

The Broadcom BCM4360 requires the proprietary `wl` driver from RPM Fusion. Fix this first — you need internet for everything else in this guide.

**1. Connect via Ethernet or USB phone tether**

- Ethernet: plug into the Gigabit port on the back of the iMac.
- Android tether: Settings → Mobile Hotspot → USB Tethering → plug USB into iMac.
- iPhone tether: Settings → Personal Hotspot → on → plug USB into iMac → tap "Trust".

Both phone methods work automatically in Fedora with zero configuration.

**2. Open Konsole**

Press `Super (⌘)` → search "Konsole" → open it.

**3. Enable RPM Fusion**

```bash
sudo dnf install \
  https://download1.rpmfusion.org/free/fedora/rpmfusion-free-release-$(rpm -E %fedora).noarch.rpm \
  https://download1.rpmfusion.org/nonfree/fedora/rpmfusion-nonfree-release-$(rpm -E %fedora).noarch.rpm
```

**4. Install Broadcom driver and reboot**

```bash
sudo dnf install broadcom-wl
sudo modprobe wl
sudo reboot
```

After reboot, the Wi-Fi icon appears in the system tray. Connect to your network normally.

> **If Wi-Fi drops constantly after connecting:**
> ```bash
> echo "blacklist b43" | sudo tee /etc/modprobe.d/blacklist-b43.conf
> echo "blacklist ssb" | sudo tee -a /etc/modprobe.d/blacklist-b43.conf
> sudo reboot
> ```

---

## 8. Phase 3 — Post-Install Essentials

**1. Update everything**

```bash
sudo dnf update -y
```

Reboot if a kernel was updated. This also updates AMDGPU firmware which may help with 5K later.

**2. Install media codecs**

```bash
sudo dnf install ffmpeg vlc
sudo dnf install @multimedia
```

**3. Enable Flathub**

```bash
flatpak remote-add --if-not-exists flathub https://dl.flathub.org/repo/flathub.flatpakrepo
```

**4. Fix Apple keyboard Fn keys**

```bash
echo "options hid_apple fnmode=2" | sudo tee /etc/modprobe.d/hid_apple.conf
sudo dracut -f
```

Makes F1–F12 the default behaviour instead of media keys. Reboot to apply.

**5. Install tools needed for the OpenCore build**

```bash
sudo dnf install curl unzip git python3 efibootmgr
```

---

## 9. Phase 4 — Reuse the USB Stick for OpenCore

> Fedora is running from the internal drive — plug the USB stick back in now.

The USB stick no longer contains the Fedora installer. You're about to wipe and reformat it as an OpenCore EFI drive. This is completely safe since Fedora runs from the internal drive.

**1. Plug the USB stick back into the iMac**

**2. Identify the USB stick's device name**

```bash
lsblk -o NAME,SIZE,FSTYPE,LABEL,TRAN
```

Look for the drive with `TRAN=usb`. It will be something like `/dev/sdb`. The internal drive is `/dev/sda` or an NVMe (`/dev/nvme0n1`).

> **Danger:** Wiping the wrong device here will destroy Fedora on your internal drive. Verify the USB device name twice before running the next commands.

**3. Wipe and format USB stick as FAT32 GPT**

```bash
sudo gdisk /dev/sdb
```

Replace `/dev/sdb` with your actual USB device. Inside gdisk:
- Type `o` → Enter (create new GPT partition table)
- Type `n` → Enter → Enter → Enter → type `EF00` → Enter (EFI partition)
- Type `w` → Enter (write and exit)

```bash
sudo mkfs.vfat -F 32 -n OPENCORE /dev/sdb1
```

The USB stick is now a clean FAT32 EFI drive ready for OpenCore.

---

## 10. Phase 4 — Build OpenCore EFI in Fedora

**1. Create the EFI directory structure**

```bash
mkdir -p ~/oc-build/EFI/OC/{Drivers,Kexts,ACPI,Tools}
mkdir -p ~/oc-build/EFI/BOOT
cd ~/oc-build
```

**2. Download OpenCore release**

Go to `github.com/acidanthera/OpenCorePkg/releases` in your browser. Download the latest `OpenCore-X.Y.Z-RELEASE.zip` (1.0.x as of early 2026). Then in Konsole:

```bash
cd ~/oc-build
unzip ~/Downloads/OpenCore-*-RELEASE.zip -d OC-release
cp OC-release/X64/EFI/OC/OpenCore.efi EFI/OC/
cp OC-release/X64/EFI/BOOT/BOOTx64.efi EFI/BOOT/
cp OC-release/X64/EFI/OC/Drivers/OpenRuntime.efi EFI/OC/Drivers/
cp OC-release/X64/EFI/OC/Tools/OpenShell.efi EFI/OC/Tools/
cp OC-release/Docs/Sample.plist EFI/OC/config.plist
```

**3. Download OpenLinuxBoot.efi and filesystem driver**

```bash
git clone https://github.com/acidanthera/OcBinaryData.git
cp OcBinaryData/Drivers/OpenLinuxBoot.efi EFI/OC/Drivers/
```

Check your Fedora root filesystem type:

```bash
df -T / | awk 'NR==2{print $2}'
```

If it prints `btrfs` (Fedora 42 default):

```bash
cp OcBinaryData/Drivers/btrfs_x64.efi EFI/OC/Drivers/
```

If it prints `ext4`:

```bash
cp OcBinaryData/Drivers/ext4_x64.efi EFI/OC/Drivers/
```

**4. Download kexts for iMac15,1**

From the following GitHub release pages, download each `.zip`, extract it, and copy the `.kext` folder into `EFI/OC/Kexts/`:

| Kext | URL |
|------|-----|
| `Lilu.kext` | `github.com/acidanthera/Lilu/releases` |
| `VirtualSMC.kext` | `github.com/acidanthera/VirtualSMC/releases` |
| `WhateverGreen.kext` | `github.com/acidanthera/WhateverGreen/releases` |
| `AirportBrcmFixup.kext` | `github.com/acidanthera/AirportBrcmFixup/releases` |

> **Note:** Each `.kext` is a folder ending in `.kext` — in Linux it appears as a directory. Keep the `.kext` extension exactly.
> Verify: `ls EFI/OC/Kexts/` should show all four `.kext` folders.

**5. Edit config.plist**

```bash
nano EFI/OC/config.plist
```

Find each key below and set the value shown. This is a Linux-only config for iMac15,1:

```
PlatformInfo > Generic > SystemProductName  →  iMac15,1

NVRAM > Add > 7C436110-... > boot-args  →
  radeon.si_support=0 radeon.cik_support=0 amdgpu.dc=1 amdgpu.mst=1

Misc > Boot > ShowPicker      →  true
Misc > Boot > HideAuxiliary  →  false
Misc > Boot > Timeout        →  5
Misc > Security > SecureBootModel  →  Disabled
Misc > Security > Vault           →  Optional
Misc > Security > ScanPolicy     →  0

Booter > Quirks > AvoidRuntimeDefrag  →  true
Kernel > Quirks > DisableIoMapper     →  true
```

In the `Drivers` array, add entries (`Enabled: true`) for:
- `OpenRuntime.efi`
- `OpenLinuxBoot.efi`
- Your filesystem driver (`btrfs_x64.efi` or `ext4_x64.efi`)

In `Kernel > Add`, add entries (`Enabled: true`) for each `.kext`. Each entry needs:
- `BundlePath` — e.g. `Lilu.kext`
- `ExecutablePath` — e.g. `Contents/MacOS/Lilu`
- `PlistPath` — `Contents/Info.plist`

> **Warning:** `config.plist` is XML. Every tag must be properly closed. A single unclosed tag makes OpenCore refuse to boot. Take your time.

**6. Validate the plist**

```bash
python3 -c "import plistlib; plistlib.load(open('EFI/OC/config.plist','rb')); print('plist XML OK')"
```

Then run the included validator for deeper checks:

```bash
chmod +x OC-release/Utilities/ocvalidate/ocvalidate
OC-release/Utilities/ocvalidate/ocvalidate EFI/OC/config.plist
```

Fix every error before continuing. Warnings are usually fine; errors are not.

---

## 11. Phase 4 — Write OpenCore to the USB Stick

**1. Mount the USB stick**

```bash
sudo mkdir -p /mnt/oc-usb
sudo mount /dev/sdb1 /mnt/oc-usb
```

Replace `/dev/sdb1` with your actual USB partition from the `lsblk` output in section 9.

**2. Copy the OpenCore EFI folder**

```bash
sudo mkdir -p /mnt/oc-usb/EFI
sudo cp -r ~/oc-build/EFI/OC /mnt/oc-usb/EFI/
sudo cp -r ~/oc-build/EFI/BOOT /mnt/oc-usb/EFI/
sudo umount /mnt/oc-usb
```

**3. Verify the structure on the USB**

```bash
sudo mount /dev/sdb1 /mnt/oc-usb
find /mnt/oc-usb -type f | sort
sudo umount /mnt/oc-usb
```

You should see `BOOTx64.efi`, `OpenCore.efi`, all your drivers, kexts, and `config.plist` listed.

OpenCore is now on the USB stick. Keep it plugged in for the next section.

---

## 12. Phase 5 — Test Boot via OpenCore USB

> USB stick plugged in — booting OpenCore from it to test.

**1. Restart and hold Option (⌥)**

At the firmware boot picker you'll see two entries: the internal Fedora GRUB and the USB stick's "EFI Boot". Select the USB stick (OpenCore).

**2. In the OpenCore picker, select Fedora**

`OpenLinuxBoot.efi` scans the internal drive and adds Fedora automatically. Select it to boot.

> **Fedora not showing in the OpenCore picker?** Press Space to reveal hidden entries. Still missing:
> 1. Check your filesystem driver is in `config.plist` as `Enabled: true`
> 2. Check `ScanPolicy` is `0`
> 3. Check `OpenLinuxBoot.efi` is `Enabled: true`
>
> Remount the USB, fix `config.plist`, unmount, reboot.

**3. After Fedora boots — check resolution**

```bash
xrandr | grep '*'
```

- Already showing `5120x2880`? The boot-args worked — skip to section 15.
- Still showing `3840x2160`? Proceed to section 14 to force 5K.

Once 5K is confirmed working, proceed to section 13 to move OpenCore to the internal EFI.

---

## 13. Phase 5 — Move OpenCore to Internal EFI (Ditch the USB)

> Do this after 5K is confirmed working via the USB stick.

This copies OpenCore from the USB to the iMac's internal EFI partition. After this step the iMac boots OpenCore directly from the internal drive — no USB needed ever again.

**1. Identify and mount the internal EFI partition**

```bash
lsblk -o NAME,SIZE,FSTYPE,LABEL,MOUNTPOINT
```

The internal EFI is the ~600 MB FAT32 partition on your internal drive — something like `/dev/sda1` or `/dev/nvme0n1p1`.

```bash
sudo mkdir -p /mnt/efi
sudo mount /dev/sda1 /mnt/efi
```

**2. Back up the current internal EFI contents**

```bash
sudo cp -r /mnt/efi/EFI ~/efi-backup-internal
```

**3. Copy OpenCore from the build directory to the internal EFI**

```bash
sudo cp -r ~/oc-build/EFI/OC /mnt/efi/EFI/
sudo cp -r ~/oc-build/EFI/BOOT /mnt/efi/EFI/
sudo umount /mnt/efi
```

**4. Register OpenCore with the firmware boot manager**

```bash
sudo efibootmgr --create --disk /dev/sda --part 1 \
  --label "OpenCore" \
  --loader "\\EFI\\BOOT\\BOOTx64.efi"
```

Adjust `--disk` and `--part` to match your internal drive. This adds "OpenCore" as a named entry in the `Option (⌥)` firmware picker.

**5. Remove the USB stick and reboot**

Unplug the USB. Restart. Hold `Option (⌥)`. You should now see "OpenCore" as an entry on the internal drive. Select it — Fedora should boot at 5K with no USB needed.

The USB stick is now completely free. Reformat it and use it for anything.

---

## 14. Phase 6 — Enable True 5120×2880

> **Always boot via OpenCore:** hold `Option (⌥)` → select OpenCore → select Fedora inside OpenCore. Booting GRUB directly bypasses OpenCore and the display drops back to 3840×2160.

### Method A — amdgpu MST (full GPU acceleration, try first)

**1. Edit GRUB**

```bash
sudo nano /etc/default/grub
```

Set `GRUB_CMDLINE_LINUX_DEFAULT` to:

```
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash radeon.si_support=0 radeon.cik_support=0 amdgpu.dc=1 amdgpu.mst=1 video=eDP-1:5120x2880@60"
```

**2. Rebuild GRUB and reboot via OpenCore**

```bash
sudo grub2-mkconfig -o /boot/grub2/grub.cfg
sudo reboot
```

**3. Verify**

```bash
xrandr | grep '*'
```

Success looks like: `5120x2880   60.00*+`

> **Warning:** Black screen after boot? Wait 15 seconds — it may revert. If it doesn't: boot into GRUB recovery mode (hold Shift at GRUB), edit the kernel line and remove `video=eDP-1:5120x2880@60`, boot. Then switch to Method B.

---

### Method B — nomodeset (most reliably confirmed, no GPU accel)

Uses the VESA framebuffer. Confirmed to produce a single clean `5120x2880` display on iMac15,1. KDE compositing is slower (CPU rendering) but browsing, LibreWolf, documents, and media playback are all fine.

**1. Set nomodeset in GRUB**

```bash
sudo nano /etc/default/grub
```

```
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash nomodeset"
```

```bash
sudo grub2-mkconfig -o /boot/grub2/grub.cfg
sudo reboot
```

**2. Verify with xrandr**

```bash
xrandr
```

Confirmed working output on iMac15,1:

```
None-1 connected 5120x2880+0+0 (normal left inverted right x axis y axis) 0mm x 0mm
```

---

### Method C — xrandr after login (amdgpu only, session fallback)

Use this if amdgpu boots successfully but initialises at 3840×2160. Needs a startup script to persist across reboots.

**1. Find your connector name**

```bash
xrandr | grep " connected"
```

**2. Add and apply the 5K mode**

```bash
xrandr --newmode "5120x2880" 1276.50 5120 5560 6128 7136 2880 2883 2888 2982 -hsync +vsync
xrandr --addmode eDP-1 "5120x2880"
xrandr --output eDP-1 --mode 5120x2880 --pos 0x0 --primary
```

Replace `eDP-1` with your actual connector name. Screen goes black? Wait 15 seconds — xrandr auto-reverts failed modes.

**3. Persist via KDE autostart script**

```bash
mkdir -p ~/.config/autostart-scripts
cat > ~/.config/autostart-scripts/5k.sh <<'EOF'
#!/bin/bash
xrandr --newmode "5120x2880" 1276.50 5120 5560 6128 7136 2880 2883 2888 2982 -hsync +vsync
xrandr --addmode eDP-1 "5120x2880"
xrandr --output eDP-1 --mode 5120x2880 --pos 0x0 --primary
EOF
chmod +x ~/.config/autostart-scripts/5k.sh
```

KDE runs all executable scripts in that folder at every login automatically.

---

## 15. Phase 6 — Set 200% HiDPI Scaling in KDE

At native 5120×2880 with 1× scaling everything is tiny — unusably so at normal desk distance. 200% scaling makes KDE render at 2× pixel density, giving a crisp "looks like 2560×1440" effective resolution. This is exactly what macOS did with this display in Retina mode.

**1. Open display settings**

System Settings → Display & Monitor → Display Configuration → Scale → set to **200%** → Apply.

KDE will prompt you to log out to apply fully. Do so.

**2. Stay on Wayland (Fedora 42 default)**

KDE Plasma 6.3 on Wayland handles 200% HiDPI perfectly with no blurring. Confirm you're on Wayland by checking the session selector at the bottom-left of the login screen — it should say "Plasma (Wayland)".

**3. Fix blurry older Flatpak/GTK apps**

```bash
sudo flatpak override --env=GDK_SCALE=2
```

Some older GTK-based Flatpak apps don't inherit Wayland scaling. This forces them to render at 2×.

> **Result:** 5120×2880 physical pixels — 2560×1440 effective working resolution — Retina-quality rendering throughout KDE Plasma.

---

## 16. Install & Configure LibreWolf

LibreWolf is a hardened Firefox fork: zero telemetry, uBlock Origin pre-installed, strict fingerprint resistance (RFP on by default), cookies cleared on close, and no built-in auto-updater. At 5K with 200% scaling, text rendering is noticeably sharper than any standard display.

### LibreWolf vs Firefox — key differences

| Feature | Firefox | LibreWolf |
|---------|---------|-----------|
| Telemetry | Opt-out | Fully removed |
| uBlock Origin | Manual install | Pre-installed |
| DRM (Netflix, Disney+) | Optional | Off by default |
| Fingerprint resistance | Basic | Strict (RFP on) |
| Cookies on close | Kept | Cleared |
| Firefox Sync | On | Off (re-enableable) |

### Method 1 — Flatpak (easiest, sandboxed)

```bash
flatpak install flathub io.gitlab.librewolf-community
```

Update with `flatpak update`. Inherits 5K/200% scaling from KDE Wayland automatically.
Install Flatseal (`flatpak install flathub com.github.tchx84.Flatseal`) if you need to grant access to paths outside your home directory.

### Method 2 — Native RPM (better KDE integration)

```bash
sudo dnf config-manager --add-repo \
  https://rpm.librewolf.net/librewolf-repo.repo
sudo dnf install librewolf
```

Updates via `sudo dnf update librewolf`. Set as default browser:

```bash
xdg-settings set default-web-browser librewolf.desktop
```

### Configure after install

**Enable DRM for streaming (Netflix, Disney+, Spotify Web)**

Address bar → `about:preferences` → scroll to "Digital Rights Management (DRM) Content" → check "Play DRM-controlled content" → restart LibreWolf. Widevine downloads automatically.

**Stay logged into sites between sessions**

Cookies are cleared on close by default — you'll be logged out everywhere on restart. To disable:
`about:preferences#privacy` → Cookies and Site Data → uncheck "Delete cookies and site data when LibreWolf is closed".

**Re-enable Firefox Sync**

`about:config` → search `identity.fxaccounts.enabled` → set to `true` → restart.

**Fix sites broken by RFP (wrong timezone, canvas errors)**

Click the LibreWolf shield icon in the address bar → toggle off "Fingerprinting Protection" for that site only. Or disable globally: `about:config` → `privacy.resistFingerprinting` → `false`.

**Enhance uBlock Origin (already active)**

Open the uBlock Origin dashboard → Filter Lists → additionally enable "AdGuard Annoyances" and "uBlock Filters — Annoyances" to block cookie consent banners and newsletter popups.

---

## 17. Set Your Wallpaper at 5K

At 5K with 200% scaling, KDE maps your wallpaper onto a 2560×1440 effective canvas. For pixel-perfect sharpness, a native 5120×2880 image is ideal — search "5K wallpaper 5120x2880" in LibreWolf. Your current 1920×1080 wallpaper will still look decent thanks to the dark edges and clean composition.

1. Save wallpaper to `~/Pictures/wallpapers/`
2. Right-click the desktop → "Configure Desktop and Wallpaper" (or System Settings → Appearance → Wallpaper)
3. Click "Add Image" → select file → set position to **"Scaled & Cropped"**
4. For best results: System Settings → Appearance → Colors → **Breeze Dark** — the dark taskbar blends seamlessly with dark-edged wallpapers

---

## 18. Troubleshooting & Known Issues

### OpenCore entry not in Option (⌥) picker

Run `sudo efibootmgr -v` and confirm the OpenCore entry exists. If not, re-run the `efibootmgr` command from section 13. If listed but not visible at boot, do a PRAM reset: hold `Cmd+Option+P+R` at startup until you see the Apple logo a second time.

### OpenCore shows but Fedora not in picker

Press Space to reveal hidden entries. Still missing:
1. Confirm your filesystem driver is in `EFI/OC/Drivers/` and has `Enabled: true` in `config.plist`
2. Confirm `ScanPolicy` is `0`
3. Confirm `OpenLinuxBoot.efi` is `Enabled: true`

Remount EFI, fix `config.plist`, unmount, reboot.

### 5K shows as split tile — two half-screens or two cursors

amdgpu is seeing two separate 2560×2880 connectors instead of a unified 5120×2880 display. Switch to Method B (`nomodeset`) in section 14 — the VESA framebuffer sees the full unified display correctly.

### Black screen after amdgpu kernel params

Wait 15 seconds — may auto-revert. If it stays black: hold Shift at GRUB to enter recovery mode, edit the kernel line, remove `video=eDP-1:5120x2880@60`, boot. Then switch to Method B (`nomodeset`).

### Booting GRUB directly drops back to 3840×2160

Expected — GRUB bypasses OpenCore and Apple's firmware degrades the display. Always boot via OpenCore: hold `Option (⌥)` → select OpenCore → select Fedora. Never boot GRUB directly if you want 5K.

### config.plist error — OpenCore panics or won't start

Enable OpenCore logging: in `config.plist` set `Misc > Debug > AppleDebug: true` and `Target: 67`. On next boot OpenCore writes a log to the EFI partition. Mount EFI and read it. Also re-run:

```bash
OC-release/Utilities/ocvalidate/ocvalidate EFI/OC/config.plist
```

### Wi-Fi drops constantly after connecting

```bash
echo "blacklist b43" | sudo tee /etc/modprobe.d/blacklist-b43.conf
echo "blacklist ssb" | sudo tee -a /etc/modprobe.d/blacklist-b43.conf
sudo reboot
```

Always reboot via OpenCore to preserve 5K.

### LibreWolf RFP breaks a site

Click the shield icon in the address bar → toggle off "Fingerprinting Protection" for that domain only.

### Upgrading Fedora 42 → 43 (EOL May 2026)

```bash
sudo dnf system-upgrade download --releasever=43
sudo dnf system-upgrade reboot
```

Trigger the upgrade reboot via OpenCore. After upgrading re-run:

```bash
sudo grub2-mkconfig -o /boot/grub2/grub.cfg
xrandr | grep '*'
```

Confirm 5K still works. RPM LibreWolf upgrades automatically with dnf. Flatpak LibreWolf upgrades independently via `flatpak update`.

---

*Guide written for iMac15,1 (27" 5K Late 2014) running Fedora KDE 42, March 2026.*
