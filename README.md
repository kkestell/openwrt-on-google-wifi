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

Next, we'll do some basic configuration of OpenWrt. We'll be setting up three pucks (`ada`, `fez`, and `rio`) in a mesh network with DAWN.

## Base Configuration

### Initial Access

SSH into the puck using its default IP:

```bash
ssh root@192.168.1.1
```

### Base Configuration for First Puck (ada)

First, set a root password:

```bash
passwd
```

Configure the hostname:

```bash
uci set system.@system[0].hostname='ada'
uci commit system
```

Set the LAN IP address:

```bash
uci set network.lan.ipaddr='192.168.1.1'
uci commit network
```

Reboot:

```bash
reboot
```

SSH back in using the new IP:

```bash
ssh root@192.168.1.1
```

Plug the WAN port into your existing network. and verify connectivity:

```bash
ping openwrt.org
```

Configure the 2.4GHz radio:

```bash
uci set wireless.radio0.disabled='0'
uci set wireless.radio0.country='US'
uci set wireless.default_radio0.ssid='S'
uci set wireless.default_radio0.encryption='psk2'
uci set wireless.default_radio0.key='topsecret'
```

Configure the 5GHz radio:

```bash
uci set wireless.radio1.disabled='0'
uci set wireless.radio1.country='US'
uci set wireless.radio1.htmode='VHT80'
uci set wireless.default_radio1.ssid='S'
uci set wireless.default_radio1.encryption='psk2'
uci set wireless.default_radio1.key='topsecret'
```

Install DAWN and its dependencies:

```bash
opkg update
opkg install wpad-wolfssl dawn luci-app-dawn
```

Enable 802.11k and band steering features:

```bash
uci set wireless.default_radio0.ieee80211k='1'
uci set wireless.default_radio0.bss_transition='1'
uci set wireless.default_radio1.ieee80211k='1'
uci set wireless.default_radio1.bss_transition='1'
```

Configure DAWN:

```bash
uci add dawn dawn
uci set dawn.@dawn[0].rssi_weight='2'
uci set dawn.@dawn[0].rssi_center='-65'
uci set dawn.@dawn[0].kick_method='1'
uci set dawn.@dawn[0].update_client='10'
```

Commit all changes and start services:
```bash
uci commit
wifi
/etc/init.d/dawn enable
/etc/init.d/dawn start
/etc/init.d/network restart
```

#### Second Puck (fez)

Follow all steps above, but use these settings instead for hostname and IP:
```bash
uci set system.@system[0].hostname='fez'
uci set network.lan.ipaddr='192.168.1.2'
uci commit
```

#### Third Puck (rio)

Follow all steps above, but use these settings instead for hostname and IP:
```bash
uci set system.@system[0].hostname='rio'
uci set network.lan.ipaddr='192.168.1.3'
uci commit
```

### Verification Steps

1. Verify wireless setup on each puck:
```bash
iwinfo
```

Should show both 2.4GHz (radio0) and 5GHz (radio1) interfaces enabled with:
- SSID: "S"
- Encryption: WPA2 PSK
- Both radios active and broadcasting

2. Verify DAWN operation:
```bash
# Check DAWN service status
/etc/init.d/dawn status

# Check logs
logread | grep dawn

# Verify umdns is running
/etc/init.d/umdns status
```

3. Access DAWN dashboard in LuCI:
- http://192.168.1.1/cgi-bin/luci/admin/dawn/network_overview
- http://192.168.1.2/cgi-bin/luci/admin/dawn/network_overview
- http://192.168.1.3/cgi-bin/luci/admin/dawn/network_overview

## Troubleshooting

If nodes aren't seeing each other:
1. Check DAWN network connectivity:
```bash
tcpdump -i br-lan port 1025
```

2. Verify all pucks have matching DAWN configurations:
```bash
uci show dawn
```

3. If umdns fails to start:
```bash
logread | grep umdns
```
