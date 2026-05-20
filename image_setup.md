# DietPi OS Cloning, Dual-Storage & Zero-IP Wi-Fi Guide
### For Raspberry Pi Zero 1 and Raspberry Pi Zero 2

This guide explains how to clone an active DietPi system to a small 6GB SSD partition, use a 64MB MicroSD card for booting, set up a 232GB exFAT partition for MPD music, and implement a **Zero-IP Wi-Fi automation** that allows mobile clients to connect via `dietpi.local` without knowing the IP address.

---

## Prerequisites
* **Raspberry Pi Zero 1** (ARMv6 32-bit) OR **Raspberry Pi Zero 2** (ARMv8 64-bit).
* **External SSD** (e.g., 240GB) connected via a microUSB-OTG adapter.
* **A dedicated small MicroSD card** (e.g., 64MB) formatted to **FAT/FAT16** labeled `boot`.
* **An active, configured DietPi system** running on a temporary SD card with network access (**SSH enabled**).

---

## Step 1: Prepare the 64MB Boot SD Card (On Windows/PC)
Since the Pi reads initial boot files from the 64MB card but runs the OS from the SSD, we must prepare this card on your PC.

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

## Step 4: Install mDNS & Update the File System Table (On the Pi via SSH)
Before shutting down, we must enable the IP-less local hostname service and tell the cloned OS where to mount its components.

1. **Install Avahi for `dietpi.local` hostname support:**
   ```bash
   apt update && apt install avahi-daemon -y
   systemctl enable --now avahi-daemon
   ```
2. Open the file system table of the cloned OS located on the SSD:
   ```bash
   nano /mnt/target/etc/fstab
   ```
3. Modify the main entries to match the new hardware mapping:
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
4. Save and exit (**Ctrl+O**, **Enter**, **Ctrl+X**).
5. Unmount the target folder and cleanly shut down the Pi:
   ```bash
   umount /mnt/target
   poweroff
   ```

---

## Step 5: First Boot and Music Storage Setup
1. Remove the temporary setup SD card from the Pi.
2. Insert your modified **64MB boot SD card**. Ensure the SSD is connected to the microUSB **Data** port. Power on the Pi.
3. Once the Pi boots up and appears on the network, log back in via SSH using `root@dietpi.local` (No IP needed!).
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
8. Link the music storage to the MPD directory:
   ```bash
   ln -s /media/usb0 /var/lib/mpd/music/SSD_Music
   dietpi-services start
   ```

---

## Bonus: Wi-Fi Automation Setup
To allow end-users to change Wi-Fi settings on a PC without touching native configuration files, create these two files directly in the root folder of the **64MB boot SD card**.

### File 1: `wifi_setup.txt` (User Interface)
```text
# THE HOME WI-FI NETWORK NAME
WIFI_NEV="Your_Wifi_Name"

# THE WI-FI PASSWORD
WIFI_JELSZO="Your_Wifi_Password"

# IP ADDRESS CONFIGURATION (Type AUTO or a static IP like 192.168.1.100)
IP_CIM=AUTO
```

### File 2: `Automation_Custom_PreScript.sh` (Background Processor)
```bash
#!/bin/bash
source /boot/firmware/wifi_setup.txt 2>/dev/null || source /boot/wifi_setup.txt

D_TXT="/boot/firmware/dietpi.txt"; [ ! -f "$D_TXT" ] && D_TXT="/boot/dietpi.txt"
W_TXT="/boot/firmware/dietpi-wifi.txt"; [ ! -f "$W_TXT" ] && W_TXT="/boot/dietpi-wifi.txt"

if [ -f "$D_TXT" ] && [ -f "$W_TXT" ]; then
    sed -i "s/^AUTO_SETUP_NET_ETHERNET_ENABLED=.*/AUTO_SETUP_NET_ETHERNET_ENABLED=0/" "$D_TXT"
    sed -i "s/^AUTO_SETUP_NET_WIFI_ENABLED=.*/AUTO_SETUP_NET_WIFI_ENABLED=1/" "$D_TXT"

    if [ "$IP_CIM" = "AUTO" ] || [ -z "$IP_CIM" ]; then
        sed -i "s/^AUTO_SETUP_NET_USESTATIC=.*/AUTO_SETUP_NET_USESTATIC=0/" "$D_TXT"
    else
        CALC_GW=$(echo "$IP_CIM" | awk -F. '{print $1"."$2"."$3".1"}')
        sed -i "s/^AUTO_SETUP_NET_USESTATIC=.*/AUTO_SETUP_NET_USESTATIC=1/" "$D_TXT"
        sed -i "s/^AUTO_SETUP_NET_STATIC_IP=.*/AUTO_SETUP_NET_STATIC_IP=$IP_CIM/" "$D_TXT"
        sed -i "s/^AUTO_SETUP_NET_STATIC_MASK=.*/AUTO_SETUP_NET_STATIC_MASK=255.255.255.0/" "$D_TXT"
        sed -i "s/^AUTO_SETUP_NET_STATIC_GATEWAY=.*/AUTO_SETUP_NET_STATIC_GATEWAY=$CALC_GW/" "$D_TXT"
        sed -i "s/^AUTO_SETUP_NET_STATIC_DNS=.*/AUTO_SETUP_NET_STATIC_DNS=1.1.1.1/" "$D_TXT"
    fi

    sed -i "s/^aWIFI_SSID\[0\].*/aWIFI_SSID=$WIFI_NEV/" "$W_TXT"
    sed -i "s/^aWIFI_KEY\[0\].*/aWIFI_KEY=$WIFI_JELSZO/" "$W_TXT"
fi
```
*Note: This script persists on the boot card and runs during every boot cycle, letting users move the device between networks seamlessly using only `dietpi.local` to connect.*
