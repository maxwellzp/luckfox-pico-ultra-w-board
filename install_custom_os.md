# Building and Installing Custom Firmware with the Luckfox Pico SDK

**Board:** Luckfox Pico Ultra W  
**Storage:** eMMC  
**Host OS:** Ubuntu 24.04 LTS  
**Build system:** Luckfox Pico SDK + Docker + Buildroot  
**SDK changes:** None in this guide

This guide shows how to build the standard firmware for a **Luckfox Pico Ultra W with eMMC** using the official Luckfox Pico SDK, without modifying the firmware configuration, and then flash the resulting image to the board.

> **Important:** Select the configuration for **RV1106_Luckfox_Pico_Ultra + EMMC + Buildroot**. This guide is not for other Luckfox Pico variants or boot media.

---

## 1. Prerequisites

You need:

- Ubuntu Linux
- Git
- Docker
- Internet access
- Luckfox Pico Ultra W with eMMC
- USB data cable
- Ethernet cable

Check Docker:

```bash
docker --version
```

---

## 2. Clone the Luckfox Pico SDK

Clone the official SDK repository:

```bash
cd ~
git clone https://github.com/LuckfoxTECH/luckfox-pico.git
cd ~/luckfox-pico
```

Inspect it:

```bash
ls -lah
```

Important SDK components include:

```text
build.sh
project/
sysdrv/
tools/
media/
rkflash.sh
```

- `project/` contains project and board configuration.
- `sysdrv/` contains system-level components such as Buildroot and the kernel.
- `tools/` contains build and packaging tools.
- `build.sh` is the main SDK build script.
- `rkflash.sh` is the SDK flashing helper.

---

## 3. Start the Luckfox Docker environment

Pull the Luckfox build image:

```bash
docker pull luckfoxtech/luckfox_pico:1.0
```

Check it:

```bash
docker images luckfoxtech/luckfox_pico
```

Create the build container:

```bash
docker run -it     --name luckfox     --privileged     -v /home/$USER/luckfox-pico:/home     luckfoxtech/luckfox_pico:1.0     /bin/bash
```

The bind mount means:

```text
Ubuntu host:  ~/luckfox-pico
Docker:       /home
```

The SDK remains on the host, while compilation takes place inside the Docker environment.

> `--privileged` gives the container extended privileges. Use this with the trusted Luckfox-provided build image.

---

## 4. Select the board configuration

Inside Docker:

```bash
cd /home
./build.sh lunch
```

Select:

```text
[5] RV1106_Luckfox_Pico_Ultra
[0] EMMC
[0] Buildroot
```

The SDK should report a configuration similar to:

```text
BoardConfig-EMMC-Buildroot-RV1106_Luckfox_Pico_Ultra-IPC.mk
```

This selection determines the target hardware, boot medium, Buildroot configuration, kernel, bootloader, partition layout, and firmware packaging.

---

## 5. Build the firmware

Run:

```bash
./build.sh
```

The first build can take some time.

Near the end, a successful build should contain messages such as:

```text
Make firmware OK!
------ OK ------
New image generated successfully!
[build.sh:info] Running build_updateimg succeeded.
[build.sh:info] Running build_firmware succeeded.
[build.sh:info] Running build_all succeeded.
[build.sh:info] Running build_save succeeded.
[build.sh:info] Running build_allsave succeeded.
```

The SDK builds the bootloader, kernel, Buildroot root filesystem, and other components and then packages them into a Rockchip `update.img`.

---

## 6. Locate the generated firmware

The individual images are generated under:

```text
output/image/
```

For example:

```text
output/image/
├── boot.img
├── download.bin
├── env.img
├── idblock.img
├── oem.img
├── rootfs.img
├── uboot.img
├── update.img
└── userdata.img
```

The complete image used by the normal SDK flashing procedure is:

```text
output/image/update.img
```

The SDK also creates a timestamped directory under:

```text
IMAGE/
```

for example:

```text
IMAGE/IPC_EMMC_BUILDROOT_RV1106_LUCKFOX_PICO_ULTRA_20260831.2239_RELEASE_TEST/
```

The timestamp will differ for each build.

---

## 7. Exit Docker

When the build is complete:

```bash
exit
```

All generated files remain on the Ubuntu host because the SDK directory was mounted into the container.

If you later want to reuse the existing container:

```bash
docker start -ai luckfox
```

Then:

```bash
cd /home
```

---

## 8. Put the board into Maskrom mode

Disconnect USB from the board.

Press and hold the **BOOT** button.

While holding BOOT, connect the board to the PC using the USB data cable.

After a few seconds, release BOOT.

Check:

```bash
lsusb
```

You should see:

```text
ID 2207:110c Fuzhou Rockchip Electronics Company
```

For this workflow, `2207:110c` is the expected Rockchip Maskrom USB device.

> If `2207:110c` is not present, do not start flashing yet. Check the USB cable, USB port, board power, and BOOT-button procedure.

---

## 9. Flash the SDK-built firmware

On the Ubuntu host:

```bash
cd ~/luckfox-pico
```

Flash the complete image:

```bash
sudo ./rkflash.sh update
```

A successful operation ends with:

```text
Download Image... (100%)
Download Firmware Success
Upgrade firmware ok.
```

### What is `rkflash.sh` doing?

You do not normally need to run `upgrade_tool` manually here.

The SDK's:

```text
rkflash.sh
```

script invokes the Rockchip upgrade tool and uses the firmware generated by the SDK.

Conceptually:

```text
./rkflash.sh update
        ↓
Rockchip upgrade tool
        ↓
output/image/update.img
        ↓
Luckfox Pico Ultra W eMMC
```

---

## 10. Check USB after flashing

After the firmware starts, check:

```bash
lsusb
```

You may now see:

```text
ID 2207:0019 Fuzhou Rockchip Electronics Company rk3xxx
```

This is normal.

The two IDs represent different states:

```text
2207:110c  → Maskrom / flashing state
2207:0019  → board after the firmware has started
```

The board can switch from one USB identity to the other without disconnecting the cable.

---

## 11. Connect Ethernet and find the IP address

Connect an Ethernet cable to the board.

The firmware normally obtains an IP address using DHCP.

From Ubuntu:

```bash
sudo arp-scan --localnet
```

The Luckfox board may have an **unknown** or **locally administered** MAC address.

If several devices are listed and you are unsure which one is the Luckfox board, disconnect its Ethernet cable, scan again, reconnect it, and scan again.

> Make sure the Ethernet cable is actually connected. A correctly flashed board can boot normally but will not appear in an ARP scan if it has no network connection.

---

## 12. Connect using SSH

Use the IP address found above:

```bash
ssh root@<LUCKFOX_IP>
```

For example:

```bash
ssh root@192.168.1.60
```

Default credentials:

```text
Username: root
Password: luckfox
```

---

## 13. Verify the firmware

Check Buildroot:

```bash
cat /etc/os-release
```

For this SDK configuration:

```text
NAME=Buildroot
VERSION=2023.02.6
ID=buildroot
VERSION_ID=2023.02.6
PRETTY_NAME="Buildroot 2023.02.6"
```

Check the kernel:

```bash
uname -a
```

The exact build number and date depend on when you built the firmware. For example:

```text
Linux luckfox 5.10.160 #1 Mon Aug 31 21:09:05 HKT 2026 armv7l GNU/Linux
```

The changed kernel build date is a useful indication that you are running the firmware built by your local SDK.

---

# Factory image vs SDK-built image

There are two different workflows.

### Factory image

```text
Official pre-built update.img
        ↓
upgrade_tool
        ↓
Luckfox eMMC
```

No SDK or Docker build environment is required.

### SDK-built image

```text
Luckfox SDK
        ↓
Docker build environment
        ↓
Buildroot + kernel + bootloader
        ↓
update.img
        ↓
rkflash.sh
        ↓
upgrade_tool
        ↓
Luckfox eMMC
```

The SDK workflow is what you use when you want to customize the firmware, for example by enabling Buildroot packages or adding your own packages.

---

# Troubleshooting

## Docker container already exists

If Docker says that the container name `luckfox` is already in use, start the existing container:

```bash
docker start -ai luckfox
```

Do not create another container unless you specifically want a separate environment.

---

## `2207:110c` is not visible

Run:

```bash
lsusb
```

If the board is missing, repeat the Maskrom procedure:

1. Disconnect USB.
2. Hold BOOT.
3. Connect USB.
4. Wait a few seconds.
5. Check `lsusb`.

You can also inspect USB kernel messages:

```bash
dmesg | tail -30
```

---

## `rkflash.sh` says no Rockchip device was found

For example:

```text
No found any rockusb device,please plug device in!
```

Check:

```bash
lsusb
```

The board should be in Maskrom mode and normally appear as:

```text
2207:110c
```

---

## `arp-scan` does not show the board

Check:

- Ethernet is connected.
- The board has finished booting.
- PC and board are on the same local network.
- The Ethernet link is active.
- You are scanning the correct network interface.

Use:

```bash
ip addr
```

to identify your active interface, then:

```bash
sudo arp-scan --localnet
```

---

## SSH reports `REMOTE HOST IDENTIFICATION HAS CHANGED`

Reflashing may change the SSH host key.

After verifying that the IP address belongs to your Luckfox board:

```bash
ssh-keygen -f ~/.ssh/known_hosts -R <LUCKFOX_IP>
```

Then connect again:

```bash
ssh root@<LUCKFOX_IP>
```

> Only remove an old SSH key after verifying that the IP address belongs to your board.

---

# Quick Reference

### Clone the SDK

```bash
git clone https://github.com/LuckfoxTECH/luckfox-pico.git
```

### Pull the Docker image

```bash
docker pull luckfoxtech/luckfox_pico:1.0
```

### Create the build container

```bash
docker run -it     --name luckfox     --privileged     -v /home/$USER/luckfox-pico:/home     luckfoxtech/luckfox_pico:1.0     /bin/bash
```

### Select the board

```bash
cd /home
./build.sh lunch
```

Select:

```text
RV1106_Luckfox_Pico_Ultra
EMMC
Buildroot
```

### Build

```bash
./build.sh
```

### Exit Docker

```bash
exit
```

### Enter Maskrom mode

Hold **BOOT** while connecting USB.

Expected:

```text
2207:110c
```

### Flash

```bash
cd ~/luckfox-pico
sudo ./rkflash.sh update
```

### Find the IP address

```bash
sudo arp-scan --localnet
```

### SSH

```bash
ssh root@<LUCKFOX_IP>
```

Default password:

```text
luckfox
```

### Verify Buildroot

```bash
cat /etc/os-release
```

### Verify kernel

```bash
uname -a
```
