# DietPi OS Cloning & Dual-Storage Setup Guide
### For Raspberry Pi Zero 1 and Raspberry Pi Zero 2

This guide explains how to clone an active, fully configured DietPi system from a temporary SD card to a small 6GB system partition on an external SSD. The setup uses a small dedicated MicroSD card (e.g., 64MB) solely for booting, leaving the rest of the SSD (~232GB) as a separate exFAT partition for MPD server music files.

---

## Prerequisites
* **Raspberry Pi Zero 1** (ARMv6 32-bit) OR **Raspberry Pi Zero 2** (ARMv8 64-bit).
* **External SSD** (e.g., 240GB) connected via a microUSB-OTG adapter.
* **A dedicated small MicroSD card** (e.g., 64MB) formatted to **FAT/FAT16** labeled `boot`.
* **An active, configured DietPi system** running on a temporary SD card with network access (**SSH enabled**).

---

## Step 1: Prepare the 64MB Boot SD Card (On Windows/PC)
Since the Pi will read initial boot files from the 64MB card but run the OS from the SSD, we must prepare this card on your PC.

1. Format the 64MB MicroSD card to **FAT (FAT16)**. Name the volume label `boot`.
2. Copy all files from your working temporary SD card's boot partition directly to the root of this 64MB card.
3. Open **`config.txt`** with a text editor (e.g., Notepad) and scroll to the very bottom:
   * 📌 **For Raspberry Pi Zero 1:** Add these lines to give the slow USB controller/SSD time to wake up:
     ```text
     boot_delay=5
     boot_delay_ms=0
     ```
   * 📌 **For Raspberry Pi Zero 2:** **Skip this step**. The Zero 2's modern bootloader automatically waits for USB drives.
4. Open **`cmdline.txt`** (ensure it remains **exactly one single line** of text) and change the `root=` parameter to point to the first partition of the USB drive:
   ```text
   root=/dev/sda1 rootfstype=ext4 rootwait fsck.repair=yes net.ifnames=0 logo.nologo console=serial0,115200 console=tty1
   ```
5. Safely eject the 64MB card from your PC.

---

## Step 2: Wipe and Partition the SSD (On the Active Pi via SSH)
Connect your SSD to the running Raspberry Pi, log in via SSH as `root`, and disable the aggressive auto-mounter before repartitioning.

1. Stop all background DietPi services (like MPD) that might lock files on the SSD:
   ```bash
   dietpi-services stop
   ```
2. Forcefully unmount any automatically mounted partitions on the SSD (verify drive names with `lsblk` first):
   ```bash
   umount -f /dev/sda1
   umount -f /dev/sda2
   ```
3. Open `fdisk` to recreate the partition layout on the SSD:
   ```bash
   fdisk /dev/sda
   ```
4. Inside the `fdisk` prompt, type the following commands (press **Enter** after each):
   * `d` $\rightarrow$ then `1` (deletes old partition 1)
   * `d` $\rightarrow$ then `2` (deletes old partition 2, if prompted)
   * `n` $\rightarrow$ (create a new partition)
   * `p` $\rightarrow$ (primary type)
   * `1` $\rightarrow$ (partition number 1)
   * *First sector:* Press **Enter** to accept the default start.
   * *Last sector:* Type **`+6G`** (allocates exactly 6GB for the cloned OS).
   * `w` $\rightarrow$ (write changes and exit).

---

## Step 3: Format and Clone the System (On the Pi via SSH)
Create a clean Linux file system on the 6GB partition and copy your live system configurations into it.

1. If the background automounter captured the new partition immediately, stop services and unmount it again:
   ```bash
   dietpi-services stop
   umount -f /dev/sda1
   ```
2. Forcefully format the new 6GB partition to `ext4`:
   ```bash
   mkfs.ext4 -F /dev/sda1
   ```
3. Create a temporary mount point and mount the 6GB SSD partition:
   ```bash
   mkdir -p /mnt/target
   mount /dev/sda1 /mnt/target
   ```
4. Clone the entire live running system from the active SD card onto the SSD using `rsync` (this takes 5-15 minutes):
   ```bash
   rsync -axHAWXS --numeric-ids --info=progress2 / /mnt/target/
   ```

---

## Step 4: Update the File System Table (On the Pi via SSH)
Before shutting down, the cloned operating system must be told where to mount its components upon the next startup.

1. Open the file system table of the cloned OS located on the SSD:
   ```bash
   nano /mnt/target/etc/fstab
   ```
2. Modify the main entries to match the new hardware mapping:
   * **The Root Line (`/`):** Change the source to `/dev/sda1` for both Pi models:
     ```text
     /dev/sda1 / ext4 noatime,lazytime,rw 0 1
     ```
   * **The Firmware Line (`/boot/firmware`):**
     * 📌 **For Raspberry Pi Zero 1:** Use the fixed hardware path:
       ```text
       /dev/mmcblk0p1 /boot/firmware vfat noatime,lazytime,rw 0 2
       ```
     * 📌 **For Raspberry Pi Zero 2:** Because of kernel differences, using the card's specific `PARTUUID` is highly recommended. Find the 64MB card's PARTUUID via `blkid /dev/mmcblk0p1` and format it like this:
       ```text
       PARTUUID=xxxxxx-01 /boot/firmware vfat noatime,lazytime,rw 0 2
       ```
3. Save and exit (**Ctrl+O**, **Enter**, **Ctrl+X**).
4. Unmount the target folder and cleanly shut down the Pi:
   ```bash
   umount /mnt/target
   poweroff
   ```

---

## Step 5: First Boot and Music Storage Setup
1. Remove the temporary setup SD card from the Pi.
2. Insert your modified **64MB boot SD card**. Ensure the SSD is connected to the microUSB **Data** port. Power on the Pi.
3. Once the Pi boots up and appears on the network, log back in via SSH as `root`.
4. Run `fdisk` once more to allocate the remaining ~232GB of unallocated space for your music files:
   ```bash
   fdisk /dev/sda
   ```
   * Type `n` $\rightarrow`p` $\rightarrow `2` $\rightarrow$ **Enter** (default start) $\rightarrow$ **Enter** (default end to take all remaining space) $\rightarrow$ `w`.
5. Install the exFAT file system tools:
   ```bash
   apt update && apt install exfat-fuse exfatprogs -y
   ```
6. Format the massive new partition to `exFAT`:
   ```bash
   mkfs.exfat /dev/sda2
   ```
7. Reboot the Pi. The automated DietPi background system will cleanly mount it under `/media/usb0`.

---

## Step 6: Link to MPD Server
To feed your massive music library smoothly into the Music Player Daemon without editing system configuration files, create a symlink:

```bash
ln -s /media/usb0 /var/lib/mpd/music/SSD_Music
dietpi-services start
```

*Everything is ready! You can now unplug the SSD anytime, plug it into your Windows/Mac PC to transfer gigabytes of music files instantly, plug it back into the Pi, and run a **Database Update** inside your MPD client application!*
