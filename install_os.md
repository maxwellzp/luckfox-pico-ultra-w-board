# Installing the Factory Firmware on Luckfox Pico Ultra W

**Board:** Luckfox Pico Ultra W  
**Storage:** eMMC  
**Host OS:** Ubuntu 24.04 LTS  
**Method:** USB / Rockchip Upgrade Tool  
**SDK required:** No

This guide describes how to install the official Buildroot firmware on a **Luckfox Pico Ultra W with eMMC** using the standalone Rockchip `upgrade_tool`.

> **Important:** This procedure writes firmware to the board's internal eMMC. Make sure you have selected an image intended for the **Luckfox Pico Ultra W eMMC** version of the board.

---

## 1. Download the official firmware

Open the official Luckfox download page:

[Official Luckfox Pico RV1106 image download](https://wiki.luckfox.com/Luckfox-Pico-RV1106/Getting-Started/overview#22-official-image-download)

For this guide, download the eMMC image:

`Luckfox_Pico_Ultra_W_EMMC_250607.zip`

The image used in this example is the factory firmware released on **June 9, 2025**.

After downloading:

```bash
cd ~/Downloads
ls -lh Luckfox_Pico_Ultra_W_EMMC_250607.zip
```

---

## 2. Extract the firmware archive

```bash
unzip Luckfox_Pico_Ultra_W_EMMC_250607.zip
```

The archive contains the firmware images and update files:

```text
Luckfox_Pico_Ultra_W_EMMC_250607/
├── boot.img
├── download.bin
├── env.img
├── .env.txt
├── idblock.img
├── oem.img
├── rootfs.img
├── sd_update.txt
├── tftp_update.txt
├── uboot.img
├── update.img
└── userdata.img
```

The most important file for this procedure is:

```text
update.img
```

`update.img` is the complete firmware image used by `upgrade_tool`.

Inspect the extracted directory:

```bash
ls -lah Luckfox_Pico_Ultra_W_EMMC_250607/
```

---

## 3. Download Rockchip `upgrade_tool`

Download the Linux version of `upgrade_tool` from the official Luckfox website:

[Download upgrade_tool v2.17](https://wiki.luckfox.com/assets/files/upgrade_tool_v2.17-bfd48dcdba9fd8013872ca2abff19a8d.zip)

Check the downloaded archive:

```bash
cd ~/Downloads
ls -lh upgrade_tool_v2.17-bfd48dcdba9fd8013872ca2abff19a8d.zip
```

Extract it:

```bash
unzip upgrade_tool_v2.17-bfd48dcdba9fd8013872ca2abff19a8d.zip
```

This creates a directory similar to:

```text
upgrade_tool_v2.17_for_linux/
├── config.ini
├── revision.txt
├── upgrade_tool
└── ...
```

---

## 4. Install `upgrade_tool`

Copy the executable to `/usr/local/bin`:

```bash
cd ~/Downloads/upgrade_tool_v2.17_for_linux
sudo cp upgrade_tool /usr/local/bin/
```

Make sure it is executable:

```bash
sudo chmod +x /usr/local/bin/upgrade_tool
```

Verify the installation:

```bash
upgrade_tool -v
```

Expected output:

```text
Upgrade Tool v2.17
```

The `upgrade_tool` is now available system-wide and can be executed from any directory.

---

## 5. Put the board into Maskrom mode

Disconnect the USB cable from the Luckfox board.

Press and hold the **BOOT** button on the board.

While holding the BOOT button, connect the board to the PC using the USB cable.

Keep the BOOT button pressed for a few seconds, then release it.

Check the connected USB devices:

```bash
lsusb
```

Look for:

```text
ID 2207:110c Fuzhou Rockchip Electronics Company
```

For this board and flashing procedure, `2207:110c` is the expected Rockchip Maskrom USB device.

> If the board does not appear as `2207:110c`, do not start flashing yet. Check the USB cable, USB port, BOOT-button procedure, and board power.

---

## 6. Check that `upgrade_tool` can see the board

Run:

```bash
sudo upgrade_tool LD
```

Expected output is similar to:

```text
List of rockusb connected(1)
DevNo=1    Vid=0x2207,Pid=0x110c,LocationID=...
```

The important part is:

```text
Pid=0x110c
```

If the tool reports:

```text
List of rockusb connected(0)
```

the board is not currently available to `upgrade_tool`. Check the USB connection and Maskrom mode before continuing.

---

## 7. Flash the factory firmware

Use the `update.img` from the extracted factory firmware directory:

```bash
sudo upgrade_tool uf ~/Downloads/Luckfox_Pico_Ultra_W_EMMC_250607/update.img
```

The tool should display progress similar to:

```text
Loading firmware...
Support Type:1106
FW Ver:0.0.00
FW Time:2025-06-09 17:16:55
Loader ver:1.01
Loader Time:2025-06-09 17:15:23

Start to upgrade firmware...
Download Boot Start
Download Boot Success
Wait For Maskrom Start
Wait For Maskrom Success
Test Device Start
Test Device Success
Check Chip Start
Check Chip Success
Get FlashInfo Start
Get FlashInfo Success
Prepare IDB Start
Prepare IDB Success
Download IDB Start
Download IDB Success
Download Firmware Start
Download Image... (100%)
Download Firmware Success
Upgrade firmware ok.
```

The most important message is:

```text
Upgrade firmware ok.
```

At this point, the factory firmware has been written to the board's eMMC.

---

## 8. Check the USB device after flashing

After a successful firmware update, the board may re-enumerate as a different USB device.

Check:

```bash
lsusb
```

You may see:

```text
ID 2207:0019 Fuzhou Rockchip Electronics Company rk3xxx
```

This is normal. The board can change its USB identity after the firmware has been loaded.

Do not confuse this with the initial Maskrom device:

```text
2207:110c
```

---

## 9. Connect Ethernet

Connect an Ethernet cable from the Luckfox Pico Ultra W to your local network.

The factory firmware uses DHCP, so the board should normally receive an IP address from your router.

Find devices on the local network:

```bash
sudo arp-scan --localnet
```

The Luckfox board may appear with an unknown or locally administered MAC address.

If you are unsure which IP belongs to the board, temporarily disconnect the Ethernet cable, run `arp-scan` again, reconnect the cable, and scan again.

> The IP address assigned by DHCP can change between boots. Do not assume that the board will always use the same address.

---

## 10. Connect to the board using SSH

Once you have identified the board's IP address, connect as `root`.

For example:

```bash
ssh root@192.168.1.53
```

The factory firmware uses:

```text
Username: root
Password: luckfox
```

On the first connection, SSH will ask you to confirm the host key. Enter:

```text
yes
```

After successful login, you should see the Luckfox shell.

---

## 11. Verify the installed operating system

Check `/etc/os-release`:

```bash
cat /etc/os-release
```

Expected output for the factory image used in this guide includes:

```text
NAME=Buildroot
VERSION=-g8e7cb31e7-dirty
ID=buildroot
VERSION_ID=2023.02.6
PRETTY_NAME="Buildroot 2023.02.6"
```

Check the kernel:

```bash
uname -a
```

For the factory image used in this guide, the output is similar to:

```text
Linux luckfox 5.10.160 #13 Mon Jun 9 17:15:47 CST 2025 armv7l GNU/Linux
```

This confirms that the board is running the expected **Buildroot 2023.02.6** factory firmware on the RV1106 ARM platform.

---

# Troubleshooting

## `upgrade_tool` cannot find the board

If:

```bash
sudo upgrade_tool LD
```

returns:

```text
List of rockusb connected(0)
```

check:

```bash
lsusb
```

The board should normally be visible as:

```text
2207:110c
```

Try the following:

1. Disconnect the USB cable.
2. Wait a few seconds.
3. Press and hold the **BOOT** button.
4. Connect the USB cable while holding BOOT.
5. Wait a few seconds.
6. Check `lsusb` again.
7. Run `sudo upgrade_tool LD`.

A complete USB disconnect and reconnect can also help if the board or USB controller is in an unexpected state.

---

## `lsusb` shows `2207:0019`, not `2207:110c`

This can happen after the firmware has been loaded and the board has left the initial Maskrom state.

For a fresh flashing operation, put the board back into Maskrom mode using the BOOT-button procedure and check for:

```text
2207:110c
```

---

## SSH reports `REMOTE HOST IDENTIFICATION HAS CHANGED`

This can happen after reflashing the board because its SSH host key may have changed.

For example:

```text
WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!
```

Remove the old key for that IP address:

```bash
ssh-keygen -f ~/.ssh/known_hosts -R 192.168.1.53
```

Then connect again:

```bash
ssh root@192.168.1.53
```

SSH will ask you to accept the new host key.

> Only remove the old key after verifying that the IP address now belongs to your Luckfox board.

---

## `arp-scan` does not show the board

Make sure:

- Ethernet is connected to the Luckfox board.
- The board has finished booting.
- The PC and Luckfox are on the same local network.
- The Ethernet link is active.
- You are scanning the correct network interface.

Run:

```bash
ip addr
```

to identify your active Ethernet interface, then:

```bash
sudo arp-scan --localnet
```

---

# What was installed?

This procedure installs the **official factory Buildroot firmware** supplied by Luckfox.

It does **not** require:

- the Luckfox SDK;
- Docker;
- Buildroot compilation;
- a cross-compiler;
- source-code changes.

The standalone `upgrade_tool` is used to write the already-built `update.img` firmware image to the board's eMMC.

The SDK is only necessary when you want to **build or customize the firmware yourself**.

---

# Quick Reference

### Download and extract the firmware

```bash
cd ~/Downloads
unzip Luckfox_Pico_Ultra_W_EMMC_250607.zip
```

### Install `upgrade_tool`

```bash
sudo cp upgrade_tool /usr/local/bin/
sudo chmod +x /usr/local/bin/upgrade_tool
```

### Check the tool

```bash
upgrade_tool -v
```

### Check the USB device

```bash
lsusb
```

Expected Maskrom device:

```text
2207:110c
```

### Check the Rockchip device

```bash
sudo upgrade_tool LD
```

### Flash the firmware

```bash
sudo upgrade_tool uf ~/Downloads/Luckfox_Pico_Ultra_W_EMMC_250607/update.img
```

### Find the board on the network

```bash
sudo arp-scan --localnet
```

### SSH

```bash
ssh root@<LUCKFOX_IP>
```

Default credentials:

```text
Username: root
Password: luckfox
```

### Verify Buildroot

```bash
cat /etc/os-release
uname -a
```
