# TL-WN727N-monitor-injection-guide
Guide + patch for enabling monitor mode &amp; packet injection on RTL8188EUS-based TP-Link adapters.
This repository documents the fix required to enable monitor mode and packet injection on certain TP-Link USB Wi-Fi adapters that use the Realtek RTL8188EUS chipset.
These adapters often fail to enter monitor mode or inject frames using the default Linux drivers.

This guide is not a driver fork.
It documents the steps, patches, and configuration changes needed to make the existing open-source driver work correctly.

Supported Devices

This method applies to any adapter showing Realtek IDs similar to:
```
lsusb | grep -i realtek
ID 2357:0111  TP-Link (RTL8188EUS)
```

Confirmed working on:

TP-Link TL-WN722N v5, v5.1, v5.21

TP-Link TL-WN727N (some revisions)

Any Realtek USB adapter using RTL8188EU / RTL8188EUS

If your device reports vendor/product ID 2357:0111 or another RTL8188EU/EUS family ID, this fix is very likely applicable.

Why It Fails Out of the Box

Linux loads the generic rtl8xxxu driver, which does not support injection or proper monitor mode for RTL8188EUS devices.

Even when using the dedicated rtl8188eus driver, newer hardware revisions (like TL-WN722N v5.21) are not fully recognized.
They require a missing flag in the USB device table.

This guide fixes that issue.

Step 1 — Install Requirements
```
sudo apt update
sudo apt install build-essential dkms git linux-headers-$(uname -r)
```
Step 2 — Clone the Driver
```
cd ~/projects/wifi
git clone https://github.com/SimplyCEO/rtl8188eus.git
cd rtl8188eus
```
Step 3 — Confirm Your Device ID (Important)

Before editing anything, confirm your adapter’s USB ID:
```
lsusb | grep -i realtek
```

Example output:

Bus 003 Device 012: ID 2357:0111 Realtek 802.11n NIC


This ID must be present in the driver source’s USB table.

Step 4 — Fix the USB Device Table

Edit:
```
os_dep/linux/usb_intf.c
```

Search for the entry:
```
{USB_DEVICE(0x2357, 0x0111)}, /* TP-Link TL-WN722N v5.21 */
```

Modify it to:
```
{USB_DEVICE(0x2357, 0x0111), .driver_info = RTL8188E}, /* TL-WN722N v5.21 */
```

This missing flag is the crucial fix.
Without .driver_info = RTL8188E, the driver initializes incorrectly and fails silently.

Step 5 — (Optional but Helpful) Update Makefile Settings

Inside the Makefile, adjust:
```
CONFIG_POWER_SAVING = n
CONFIG_USB_AUTOSUSPEND = n
# Optional debugging:
CONFIG_RTW_DEBUG = y
```

These options prevent autosuspend issues and make debugging easier.

Step 6 — Install Firmware (If Missing)

Your repo may not include firmware. Install it manually:
```
cd /tmp
wget https://github.com/lwfinger/rtl8188eu/raw/v5.3.9/rtl8188eufw.bin
sudo cp rtl8188eufw.bin /lib/firmware/
sudo cp rtl8188eufw.bin /lib/firmware/rtlwifi/
```
Step 7 — Build & Install the Driver
```
make clean
make -j$(nproc)
sudo make install
```
Step 8 — Blacklist the Conflicting Driver
```
echo "blacklist rtl8xxxu" | sudo tee /etc/modprobe.d/rtl8xxxu-blacklist.conf
sudo update-initramfs -u
```
Step 9 — Load the Correct Driver
```
sudo modprobe --remove rtl8xxxu
sudo modprobe 8188eu
```

Verify:
```
lsmod | grep 8188eu
```
Step 10 — Replug Adapter & Verify Recognition
```
lsusb | grep -i realtek
ip link show
iwconfig
```

You should now see a new interface (e.g., wlan1 or wlx...).

Step 11 — Enable Monitor Mode

Replace wlan1 with your actual interface name:
```
sudo ip link set wlan1 down
sudo iw wlan1 set monitor control
sudo ip link set wlan1 up
```

Verify:
```
iwconfig wlan1
```

Should show:
```
Mode:Monitor
```
Step 12 — Test Packet Injection
```
sudo aireplay-ng --test wlan1
```

A successful output looks like:
```
Injection is working!
Found X APs
```

If you see injection rates (like 100%, 40%, 0%), the driver is functioning exactly as intended.

Recompiling After Kernel Updates

Every major kernel update requires rebuilding:
```
cd ~/projects/wifi/rtl8188eus
make clean
make -j$(nproc)
sudo make install
sudo modprobe -r 8188eu
sudo modprobe 8188eu
```
Patch File (For Contributors)

If you want to apply the USB fix automatically, create a file like:
```
rtl8188eus-tlwn722n-v5.patch
--- a/os_dep/linux/usb_intf.c
+++ b/os_dep/linux/usb_intf.c
@@ -123,7 +123,7 @@ static struct usb_device_id rtw_usb_id_tbl[] = {
-    {USB_DEVICE(0x2357, 0x0111)},
+    {USB_DEVICE(0x2357, 0x0111), .driver_info = RTL8188E}, /* TL-WN722N v5.21 fix */
```

Users can apply it using:
```
patch -p1 < rtl8188eus-tlwn722n-v5.patch
```
Credits

Original driver source: SimplyCEO / Realtek contributors

Firmware: lwfinger’s Realtek firmware repo

Additional debugging & testing notes from my own documentation
Legal / Ethical Note

Monitor mode and packet injection should be used only on networks you own or have explicit permission to test.
Legal / Ethical Note Monitor mode and packet injection should be used only on networks you own or have explicit permission to test.
