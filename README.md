# Overview

Google Wifi (AC-1304) is a 2x2 802.11ac access point with a Qualcomm IPQ4019 SoC, 512 MB RAM, and 4 GB eMMC storage. It runs a ChromeOS-based firmware called Gale and can be configured to boot unsigned firmware like OpenWrt by enabling Developer Mode.

# Installation

The high level steps to install OpenWrt on Google Wifi are as follows:

1. Flash the official firmware to the device.
2. Boot OpenWrt from a USB drive.
3. Flash OpenWrt to the device's eMMC.
4. Flash the sysupgrade image to the device.

Shoutout to [this post](https://forum.openwrt.org/t/finally-installed-openwrt-on-my-google-wifi-ac-1304/183541/2) by papdee. This guide is based on their instructions.

## Hardware Requirements

Obviously, you'll need a Google Wifi AC-1304.

You'll also need the following:

- **USB-C hub with power delivery (PD)**
  
  I used an [Anker 341 USB-C Hub (7-in-1)](https://www.anker.com/products/a8346). The important thing is that it has a USB-C PD port for power and data passthrough to the puck, as well as a USB-A port for the USB drive.

- **USB-C power supply**

  I used a 30W USB-C charger from an iPad.

- **USB-C cable**
  
  I used the cable that came with my iPhone.

- **2 USB 3.0 flash drives**

  I used a pair of 32 GB USB 3.0 drives from Micro Center. It's strongly recommended that you use drives with activity LEDs to monitor the flashing process.

- **Phillips PH0 screwdriver**

  The puck's bottom cover is held on by a single screw.

- **Plastic spudger or flat-head screwdriver**

  To pry off the bottom cover.

## Prepare the Official Firmware USB Drive

1. Install the [OnHub Recovery Utility](https://chromewebstore.google.com/detail/onhub-recovery-utility/fmgkgdalfapcmjnanilfcpkhkhedmpdm?pli=1) in Chrome.
2. Select "Google" as the manufacturer and "Google Wifi" as the model.
3. Write the latest official Google Wifi firmware to a USB drive.

## Flash the Official Firmware

1. Connect the USB-C hub to power but do not plug it into the puck yet.
2. Insert the firmware USB drive into the hub.
3. Hold the external reset button on the puck.
4. Plug in the USB-C hub while continuing to hold the button.
5. After ~10 seconds, the LED will turn amber.
6. Release the button and wait ~5 minutes for the process to complete.
7. When the LED glows solid blue, the firmware has been installed.

## Prepare the OpenWrt USB Drive

1. Download the latest OpenWrt "factory" image for Google Wifi from the [OpenWrt website](https://openwrt.org/toh/google/wifi).
2. Use the OnHub Recovery Utility to write this image to a second USB drive.
   - Open the utility, click the gear icon, select "Use local image," and choose the OpenWrt factory image.

## Open the Puck and Locate SW7

1. Disconnect power from the puck.
2. Remove the bottom cover by unscrewing the Phillips screw and prying it off.
3. Locate the internal switch labeled "SW7."

## Boot OpenWrt

1. Insert the OpenWrt USB drive into the powered USB-C hub.
2. Hold the external reset button while plugging in the USB-C hub.
3. After ~10 seconds, the LED will turn amber.
4. Press SW7 inside the device. The LED will blink purple, then reboot.
5. When the LED blinks purple again, press SW7 once more to trigger USB boot.
6. If the device successfully boots OpenWrt, you should be able to ping `192.168.1.1` from a computer connected via Ethernet.

If the puck continues blinking purple, try pressing SW7 again or check the USB drive.

## Flash OpenWrt to eMMC

1. Transfer the OpenWrt factory image to `/tmp` on the puck using SCP:
   ```sh
   scp -O openwrt-23.05.5-ipq40xx-chromium-google_wifi-squashfs-factory.bin root@192.168.1.1:/tmp/
   ```
2. SSH into the puck:
   ```sh
   ssh root@192.168.1.1
   ```
3. Write the firmware to eMMC:
   ```sh
   dd if=/dev/zero bs=512 seek=7634911 of=/dev/mmcblk0 count=33
   dd if=/tmp/openwrt-23.05.5-ipq40xx-chromium-google_wifi-squashfs-factory.bin of=/dev/mmcblk0
   ```
4. Reboot the device:
   ```sh
   reboot
   ```
5. Remove the USB drive. The puck will now boot OpenWrt from eMMC.

## Flash the Sysupgrade Image

1. Download the latest OpenWrt "sysupgrade" image from the OpenWrt website.
2. Access the puck's web interface at `http://192.168.1.1`.
3. Navigate to `System > Backup/Flash Firmware > Flash new firmware image`.
4. Upload the sysupgrade image and confirm.
5. The puck will reboot automatically.

## Notes

- If OpenWrt fails to boot from USB, try a different flash drive.
- If the puck remains in a purple-blink loop, ensure SW7 is pressed at the correct moment.
- The default OpenWrt user is `root` with no password. Set a password using `passwd` or in the web interface.
- If installation repeatedly fails, consider re-flashing the official firmware before retrying OpenWrt.

# Configuration

## Expand Storage

1. SSH into the puck and install required utilities:
   ```sh
   opkg update && opkg install cfdisk resize2fs
   ```
2. Run `cfdisk`:
   ```sh
   cfdisk /dev/mmcblk0
   ```
3. Resize the last partition:
   - Select the last partition before the empty space.
   - Choose "Resize."
   - Accept the default size.
   - Write the partition table and exit.
4. Reboot the device.
5. Resize the filesystem:
   ```sh
   resize2fs /dev/loop0
   ```