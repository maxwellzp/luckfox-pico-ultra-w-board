# Adding a Buildroot Package to the Luckfox Pico Ultra W Firmware

**Board:** Luckfox Pico Ultra W  
**Storage:** eMMC  
**Host OS:** Ubuntu 24.04 LTS  
**Build system:** Luckfox Pico SDK + Docker + Buildroot  
**Example package:** Git

This guide explains how to enable a software package that is already supported by Buildroot, rebuild the Luckfox firmware, flash it to the board, and verify that the package is available.

The example uses **Git**.

> **Important:** Git is already supported by the Buildroot version included in the Luckfox SDK. We are enabling an existing Buildroot package; we are not creating a new package in this guide.

---

## 1. Start the Docker container

The Luckfox SDK build environment is provided through Docker.

If the `luckfox` container already exists, start it with:

```bash
docker start -ai luckfox
```

You should enter the container:

```text
root@1ae305c0e484:/#
```

---

## 2. Go to the mounted SDK directory

The SDK is mounted at `/home` inside the container:

```bash
cd /home
```

The SDK should contain:

```text
build.sh
project/
sysdrv/
tools/
rkflash.sh
```

---

## 3. Open the Buildroot configuration menu

Run:

```bash
./build.sh buildrootconfig
```

This opens the Buildroot `menuconfig` interface for the currently selected Luckfox board configuration.

You will see sections such as:

```text
Target options  --->
Toolchain  --->
Build options  --->
System configuration  --->
Kernel  --->
Target packages  --->
Filesystem images  --->
Bootloaders  --->
Host utilities  --->
Legacy config options  --->
```

Select:

```text
Target packages  --->
```

`Target packages` contains software that can be included in the root filesystem of the embedded board.

---

## 4. Find the Git package

Buildroot's menu can be searched directly.

Press:

```text
/
```

Enter:

```text
BR2_PACKAGE_GIT
```

Select the matching configuration option.

Enable it by pressing **Space**.

The Git entry should now be selected:

```text
[*] git
```

### What does `BR2_PACKAGE_GIT` mean?

Buildroot uses configuration symbols to control which software is included in the target firmware.

For Git:

```text
BR2_PACKAGE_GIT=y
```

means:

> Build Git and include it in the target filesystem.

Without it:

```text
# BR2_PACKAGE_GIT is not set
```

Git is not included in the firmware.

---

## 5. Save the Buildroot configuration

In `menuconfig`, select:

```text
Save
```

Then exit the menu.

If Buildroot asks you to confirm the configuration changes, choose **Yes**.

Verify the active configuration:

```bash
grep '^BR2_PACKAGE_GIT'     sysdrv/source/buildroot/buildroot-2023.02.6/.config
```

Expected:

```text
BR2_PACKAGE_GIT=y
```

This confirms that Git is enabled in the current Buildroot configuration.

---

## 6. Save the configuration to the SDK defconfig

The `.config` file is the current Buildroot configuration used for the build.

The Luckfox SDK also has a board-specific defconfig:

```text
sysdrv/source/buildroot/buildroot-2023.02.6/configs/luckfox_pico_w_defconfig
```

Save the configuration there so that the Git selection becomes part of the board configuration.

Enter the Buildroot directory:

```bash
cd /home/sysdrv/source/buildroot/buildroot-2023.02.6
```

Run:

```bash
make savedefconfig
```

Then verify:

```bash
grep '^BR2_PACKAGE_GIT' configs/luckfox_pico_w_defconfig
```

Expected:

```text
BR2_PACKAGE_GIT=y
```

### Why use `savedefconfig`?

Buildroot has:

```text
.config
```

which represents the current configuration, and a defconfig which provides the compact configuration used to reproduce that setup.

For this Luckfox project, `BR2_DEFCONFIG` points to:

```text
configs/luckfox_pico_w_defconfig
```

Saving the defconfig makes the change persistent in the SDK configuration.

> **Important:** Run `make savedefconfig` from the Buildroot directory. In this project that is:
>
> ```text
> /home/sysdrv/source/buildroot/buildroot-2023.02.6
> ```

---

## 7. Build the new firmware

Return to the SDK root:

```bash
cd /home
```

Build the complete firmware:

```bash
./build.sh
```

Buildroot will build Git and any required dependencies, install them into the target filesystem, and the SDK will package the complete firmware into a new `update.img`.

A successful build ends with messages similar to:

```text
Make firmware OK!
------ OK ------
...
New image generated successfully!
[build.sh:info] Running build_updateimg succeeded.
[build.sh:info] Running build_firmware succeeded.
[build.sh:info] Running build_all succeeded.
...
[build.sh:info] Running build_save succeeded.
[build.sh:info] Running build_allsave succeeded.
```

The exact timestamp in the generated `IMAGE/` directory will depend on the build time.

---

## 8. Exit Docker

When the build has completed:

```bash
exit
```

Because the SDK directory is mounted into Docker, the generated firmware remains available on the Ubuntu host.

The SDK is located at:

```text
~/luckfox-pico/
```

---

## 9. Check the board's current USB state

If the board is already connected:

```bash
lsusb | grep "Fuzhou Rockchip Electronics Company"
```

A running board may appear as:

```text
ID 2207:0019 Fuzhou Rockchip Electronics Company rk3xxx
```

This is the normal running state.

---

## 10. Put the board into Maskrom mode

Disconnect the USB cable from the board.

Press and hold the **BOOT** button.

While holding BOOT, connect the USB cable to the PC.

Wait a few seconds and release the button.

Check:

```bash
lsusb | grep "Fuzhou Rockchip Electronics Company"
```

The board should appear as:

```text
ID 2207:110c Fuzhou Rockchip Electronics Company
```

For this flashing procedure:

```text
2207:110c → Maskrom mode
```

> If `2207:110c` does not appear, do not start flashing yet. Check the USB cable, USB port, board power, and BOOT-button procedure.

---

## 11. Flash the new firmware

On the Ubuntu host:

```bash
cd ~/luckfox-pico
```

Flash the complete firmware:

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

You do not normally need to run `upgrade_tool` manually.

The SDK's:

```text
rkflash.sh
```

script invokes the Rockchip upgrade tool and uses the firmware generated by the SDK.

The workflow is:

```text
Buildroot configuration
        ↓
./build.sh
        ↓
update.img
        ↓
./rkflash.sh update
        ↓
upgrade_tool
        ↓
Luckfox Pico Ultra W eMMC
```

---

## 12. Check the USB device after flashing

Run:

```bash
lsusb | grep "Fuzhou Rockchip Electronics Company"
```

You should eventually see:

```text
ID 2207:0019 Fuzhou Rockchip Electronics Company rk3xxx
```

The transition is:

```text
2207:110c → Maskrom / flashing state
2207:0019 → normal running state
```

---

## 13. Find the board's IP address

Connect Ethernet to the Luckfox board.

The Buildroot firmware normally obtains an IP address using DHCP.

From Ubuntu:

```bash
sudo arp-scan --localnet
```

The Luckfox board may appear with an unknown or locally administered MAC address.

If you are unsure which IP address belongs to the board, disconnect its Ethernet cable, run `arp-scan` again, reconnect it, and scan again.

> The DHCP-assigned IP address can change between boots.

---

## 14. Connect using SSH

Connect to the board:

```bash
ssh root@<LUCKFOX_IP>
```

For example:

```bash
ssh root@192.168.1.38
```

Default credentials:

```text
Username: root
Password: luckfox
```

---

## 15. Verify that Git is installed

Run:

```bash
git --version
```

Expected output for the configuration used in this example:

```text
git version 2.39.3
```

You can also check where the executable is located:

```bash
which git
```

Typically:

```text
/usr/bin/git
```

This confirms that Git was included in the target root filesystem and is available on the running Luckfox board.

---

# What happened during the build?

Before enabling Git, the Buildroot configuration contains:

```text
# BR2_PACKAGE_GIT is not set
```

After enabling it:

```text
BR2_PACKAGE_GIT=y
```

Buildroot then:

1. Detects that Git is enabled.
2. Builds Git using the target toolchain.
3. Builds any required dependencies.
4. Installs Git into the target root filesystem.
5. Packages the filesystem into the firmware.
6. The Luckfox SDK creates a new `update.img`.
7. `rkflash.sh` flashes the new image to the eMMC.

The important point is that Git is **not installed directly onto the already-running board**.

Instead:

```text
Buildroot configuration
        ↓
Build Git
        ↓
Put Git into root filesystem
        ↓
Build firmware
        ↓
Flash firmware
        ↓
Git exists on the board
```

---

# Buildroot packages vs installing software on Linux

This is different from installing software on Ubuntu.

On Ubuntu you might use:

```bash
sudo apt install git
```

The embedded Buildroot system normally works differently.

Software is selected when the firmware is built:

```text
Buildroot menuconfig
        ↓
BR2_PACKAGE_GIT=y
        ↓
Build firmware
        ↓
Flash firmware
```

The resulting firmware contains Git.

This is one of the important differences between a general-purpose Linux distribution and an embedded Buildroot system.

---

# Troubleshooting

## `BR2_PACKAGE_GIT` cannot be found

Make sure you opened the Buildroot configuration for the selected Luckfox board:

```bash
./build.sh buildrootconfig
```

Search specifically for:

```text
BR2_PACKAGE_GIT
```

If it is still unavailable, verify that you are using the Buildroot tree supplied by the SDK:

```text
sysdrv/source/buildroot/buildroot-2023.02.6/
```

---

## Git is enabled but is not on the board

Verify the active configuration:

```bash
grep '^BR2_PACKAGE_GIT'     sysdrv/source/buildroot/buildroot-2023.02.6/.config
```

Expected:

```text
BR2_PACKAGE_GIT=y
```

Make sure the firmware was rebuilt after changing the configuration:

```bash
./build.sh
```

Then flash the newly generated image:

```bash
sudo ./rkflash.sh update
```

---

## `make savedefconfig` fails

Make sure you are inside the Buildroot directory:

```bash
cd /home/sysdrv/source/buildroot/buildroot-2023.02.6
```

Then:

```bash
make savedefconfig
```

You can check where Buildroot expects to save the defconfig:

```bash
grep '^BR2_DEFCONFIG' .config
```

It should point to the Luckfox board defconfig:

```text
configs/luckfox_pico_w_defconfig
```

---

## `rkflash.sh` cannot find the board

If you see:

```text
No found any rockusb device,please plug device in!
```

check:

```bash
lsusb
```

The board should be in Maskrom mode and normally appear as:

```text
2207:110c
```

Repeat the BOOT-button procedure if necessary.

---

## `arp-scan` does not show the board

Make sure:

- Ethernet is connected.
- The board has finished booting.
- The PC and board are on the same local network.
- The Ethernet link is active.
- You are scanning the correct network interface.

Check interfaces with:

```bash
ip addr
```

Then:

```bash
sudo arp-scan --localnet
```

---

## SSH reports `REMOTE HOST IDENTIFICATION HAS CHANGED`

Reflashing can change the board's SSH host key.

After verifying that the IP address belongs to your Luckfox board:

```bash
ssh-keygen -f ~/.ssh/known_hosts -R <LUCKFOX_IP>
```

Then:

```bash
ssh root@<LUCKFOX_IP>
```

> Only remove an old SSH key after verifying that the IP address belongs to your board.

---

# Quick Reference

### Start Docker

```bash
docker start -ai luckfox
```

### Enter the SDK

```bash
cd /home
```

### Open Buildroot configuration

```bash
./build.sh buildrootconfig
```

Enable:

```text
BR2_PACKAGE_GIT
```

### Verify the active configuration

```bash
grep '^BR2_PACKAGE_GIT'     sysdrv/source/buildroot/buildroot-2023.02.6/.config
```

Expected:

```text
BR2_PACKAGE_GIT=y
```

### Save the configuration

```bash
cd /home/sysdrv/source/buildroot/buildroot-2023.02.6
make savedefconfig
```

### Verify the SDK defconfig

```bash
grep '^BR2_PACKAGE_GIT' configs/luckfox_pico_w_defconfig
```

Expected:

```text
BR2_PACKAGE_GIT=y
```

### Build

```bash
cd /home
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

### Verify normal USB state

```bash
lsusb | grep "Fuzhou Rockchip Electronics Company"
```

Expected:

```text
2207:0019
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

### Verify Git

```bash
git --version
```

Expected:

```text
git version 2.39.3
```
