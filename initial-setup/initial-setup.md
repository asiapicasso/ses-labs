# Lab 1 – Buildroot and Raspberry Pi 4

---

# Part 1 – Development environment

Installed tools: **Git**, **Docker Desktop** (Apple silicon version), **VS Code** with the **Dev Containers** extension, **balenaEtcher** (or `dd`) and **picocom**.

```bash
git config --global user.name "First Last"
git config --global user.email "first.last@hes-so.ch"

git clone https://github.com/MA-SeS/resources
code resources/labs        # then "Reopen in Container"

git clone https://github.com/asiapicasso/ses-labs.git   # repository for the lab journals
```

Inside the container, we work in `/workspace` (= the Mac's `labs/` folder). Buildroot 2025.02.17 is in `/workspace/buildroot`, on the `local-lts` branch.

---

# Part 2 – Vanilla SD card image

## Configure Buildroot

```bash
cd /workspace/buildroot
ls
```

```
CHANGES           DEVELOPERS       SECURITY.md  configs  package    utils
COPYING           Makefile         arch         docs     support
Config.in         Makefile.legacy  board        fs       system
Config.in.legacy  README           boot         linux    toolchain
```

```bash
make list-defconfigs | grep raspberrypi4
make raspberrypi4_64_defconfig
make menuconfig
```

> **Question:** _What configuration option(s) gave you a strong confidence that it is indeed the right configuration for your target?_

**Answer:** the **AArch64 / Cortex-A72** architecture (the Pi 4 CPU), the **`bcm2711-rpi-4-b`** Device Tree, and the **rpi-firmware** package set to the Raspberry Pi 4 variant.

## Build

```bash
make -j$(nproc)
```

```
You have an uutils 'install' version installed which is affected by:
  https://github.com/uutils/coreutils/issues/12166

Please change to coreutils install with:
update-alternatives --install /usr/bin/install install /usr/bin/gnuinstall 100
make: *** [support/dependencies/dependencies.mk:27: dependencies] Error 1
```

The build stops immediately with a `uutils 'install'` error. This is expected: Ubuntu 26.04 ships an `install` written in Rust, which is not compatible with the GNU version. Fix:

```bash
sudo update-alternatives --install /usr/bin/install install /usr/bin/gnuinstall 100
ls -l /usr/bin/install
install --version | head -1
```

```
update-alternatives: using /usr/bin/gnuinstall to provide /usr/bin/install (install) in auto mode
lrwxrwxrwx 1 root root 25 Sep 27 18:40 /usr/bin/install -> /etc/alternatives/install
install (GNU coreutils) 9.7
```

```bash
make -j$(nproc)
```

> **Question:** _Identify each directory where the following are stored: downloaded packages (tarballs); sources and compiled files; tools for the host, in particular the cross-compilation toolchain._

**Answer:** tarballs → `dl/` · sources and compiled files → `output/build/` · host tools and toolchain → `output/host/`

> **Question:** _Beside sdcard.img, can you identify and understand the purpose of each file in output/images?_

**Answer:**

- `Image`: the Linux kernel
- `*.dtb`: Device Trees (hardware description)
- `rpi-firmware/`: boot firmware, `config.txt`, `cmdline.txt`, `overlays/`
- `boot.vfat`: the boot partition
- `rootfs.ext4`: the root filesystem

## Inspect the image

```bash
sudo apt-get install -y fdisk
fdisk -l output/images/sdcard.img
```

```
Disk output/images/sdcard.img: 152 MiB, 159384064 bytes, 311297 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0x00000000

Device                    Boot Start    End Sectors  Size Id Type
output/images/sdcard.img1 *        1  65536   65536   32M  c W95 FAT32 (LBA)
output/images/sdcard.img2      65537 311296  245760  120M 83 Linux
```

> **Question:** _How many partitions do you see, what are their offsets and sizes, and of what are their types?_

**Answer:** 2 partitions (512-byte sectors, so offset = Start × 512 and size = Sectors × 512).

| #   | Offset       | Size    | Type  |
| --- | ------------ | ------- | ----- |
| 1   | 512 B        | 32 MiB  | FAT32 |
| 2   | 33,554,944 B | 120 MiB | Linux |

> 🤖 **About the partitions:**
>
> - The image uses an **MBR (DOS)** partition table, stored in sector 0. That is why partition 1 starts at byte 512.
> - **Partition 1 (boot, FAT32):** the Pi's boot ROM can only read FAT, so the firmware, `config.txt`, the kernel and the Device Trees must live here. The `*` in fdisk marks it as bootable.
> - **Partition 2 (rootfs, Linux):** holds the Linux system mounted as `/` (`root=/dev/mmcblk0p2` in `cmdline.txt`). It starts right after partition 1 (sector 65537).
> - `mount` needs **bytes**, not sectors: hence the × 512 conversion for `offset` and `sizelimit`.

```bash
mkdir -p /tmp/p1 /tmp/p2
sudo mount -r -o loop,offset=512,sizelimit=33554432 output/images/sdcard.img /tmp/p1
sudo mount -r -o loop,offset=33554944,sizelimit=125829120 output/images/sdcard.img /tmp/p2
mount | grep /tmp/p
```

```
/workspace/buildroot/output/images/sdcard.img on /tmp/p1 type vfat (ro,relatime,fmask=0022,dmask=0022,codepage=437,iocharset=iso8859-1,shortname=mixed,errors=remount-ro)
/workspace/buildroot/output/images/sdcard.img on /tmp/p2 type ext4 (ro,relatime)
```

> **Question:** _What are the types of each filesystem?_

**Answer:** partition 1 → **vfat**, partition 2 → **ext4**.

> **Question:** _What files are present in the first partition and what is their purpose?_

**Answer:** the boot files: firmware (`start4.elf`, `fixup4.dat`), `config.txt`, `cmdline.txt`, the `Image` kernel, the Device Trees and `overlays/`.

> **Question:** _What's the contents of the second partition?_

**Answer:** the Linux rootfs (`bin`, `etc`, `lib`, `usr`…).

```bash
sudo umount /tmp/p1 /tmp/p2
```

## Flash the SD card

```bash
# [container] copy the image to labs/sd_dir
bash /workspace/resources/utilities/sd_copy.sh
```

On the Mac, with `dd` (balenaEtcher crashed with the error `requestMetadata is not a function`). The 32 GB SD card is `/dev/disk4`:

```bash
cd ~/git/resources/labs/sd_dir
diskutil unmountDisk /dev/disk4
sudo dd if=sdcard.img of=/dev/rdisk4 bs=4m status=progress
sync
diskutil eject /dev/disk4
```

```
159384064 bytes transferred in 2.23 secs (71 MB/s)
```

Check that the card contains the two partitions of the image:

```bash
diskutil list external physical
```

```
/dev/disk4 (external, physical):
   0:     FDisk_partition_scheme                        *31.9 GB    disk4
   1:             Windows_FAT_32 NO NAME                 33.6 MB    disk4s1
   2:                      Linux                         125.8 MB   disk4s2
                    (free space)                         31.8 GB    -
```

The sizes match those seen with `fdisk` (32 MiB and 120 MiB). The rest of the card is unused.

> 📌 On the Mac, `labs/buildroot` is **empty**: Buildroot lives in a Docker volume, only visible from the container. That is why we go through `sd_copy.sh` and `sd_dir`.

## Serial connection

> 🤖 **When to plug in the Raspberry Pi:** only after the SD card is flashed and ejected. Keep the Pi **unpowered**, then:
>
> 1. Insert the SD card into the Pi.
> 2. Wire the serial cable to the Pi header (see below), then plug its USB side into the Mac.
> 3. Start `picocom` on the Mac.
> 4. **Last**, power the Pi through USB-C, so the boot logs appear from the very first line.

```bash
ls /dev/cu.*                                       # [Mac] find the serial device
picocom -b 115200 /dev/cu.usbserial-XXXX           # [Mac]
```

Wiring: GND → pin 6, cable RX → pin 8 (Pi TX), cable TX → pin 10 (Pi RX).

| Couleur | Signal                | À brancher sur le Pi          |
| ------- | --------------------- | ----------------------------- |
| Noir    | GND                   | **Pin 6** (GND)               |
| Jaune   | RXD (le câble reçoit) | **Pin 8** (GPIO14, TX du Pi)  |
| Orange  | TXD (le câble envoie) | **Pin 10** (GPIO15, RX du Pi) |
| Rouge   | 5 V                   | Rien, ne pas brancher         |
| Brun    | CTS                   | Rien                          |
| Vert    | RTS                   | Rien                          |

## Checks on the target

```bash
df -h ; uname -r ; nproc ; cat /proc/meminfo
```

> **Question:** _What's the actual size of the rootfs?_

**Answer:** about 120 MiB. ✏️ `df -h`: \_\_\_\_

> **Question:** _What Linux kernel version is actually running?_

**Answer:** 6.6.28-v8

> **Question:** _How many CPUs are present?_

**Answer:** 4

> **Question:** _How much RAM is present in total and how much is actually available?_

**Answer:** ✏️ `MemTotal` = \_**\_ · `MemAvailable` = \_\_**

> **Question:** _What's the difference between MemFree and MemAvailable?_

**Answer:** MemFree is the RAM that is completely unused. MemAvailable adds the caches the kernel can free: it is the memory actually available to a program.

---

# Part 3 – Custom configuration and U-Boot

## Create your own config

```bash
cp configs/raspberrypi4_64_defconfig configs/seslab_rpi4_defconfig
make seslab_rpi4_defconfig
make menuconfig
make savedefconfig
```

## Hostname, banner, password

In **System configuration**: _System hostname_, _System banner_ and _Root password_.

> **Question:** _What setting did you change to add a password for the root user?_

**Answer:** **System configuration → Root password** (`BR2_TARGET_GENERIC_ROOT_PASSWD`).

## Switch to U-Boot

1. **Bootloaders → U-Boot**, _Board defconfig_ = `rpi_4`
2. In `board/raspberrypi/config_4_64bit.txt`: replace `kernel=Image` with `kernel=u-boot.bin`
3. In `board/raspberrypi/genimage.cfg.in`: add `"Image",` in `files = {`

```bash
rm output/images/sdcard.img output/images/boot.vfat
make rpi-firmware-reinstall
make -j$(nproc)
```

> **Question:** _Remembering that your board is a Raspberry Pi 4, specify the correct U-Boot config._

**Answer:** `rpi_4`

> **Question:** _What difference do you see in output/images/ in comparison to Part 2?_

**Answer:** a new `u-boot.bin` file appears, and `config.txt` now contains `kernel=u-boot.bin`.

## Boot into U-Boot

> 🤖 **When to plug in the Raspberry Pi:** unplug the Pi's power, flash the new image, put the SD card back in, start `picocom`, then power the Pi through USB-C.

At boot, press a key during "Hit any key to stop autoboot" to get the `U-Boot>` prompt.

---

# Key concepts of this lab

> **Question:** _Explain what you did in this lab._

**Answer:** I built a complete embedded Linux system for the Raspberry Pi 4 from source with Buildroot, inspected the image, flashed it to an SD card, booted it, then customized it and added U-Boot as bootloader.

Key points to remember:

- **Containerized environment:** the whole build runs in a Docker Dev Container (Ubuntu 26.04), so it works the same on Mac, Windows and Linux. Buildroot lives in a Docker volume, so it is only visible from the container.
- **Buildroot builds everything:** the cross-compilation toolchain, the Linux kernel, the bootloader, the root filesystem and the final `sdcard.img`, from a single configuration (`defconfig`).
- **Cross-compilation:** the code is compiled on the host (x86/ARM Mac) for a different target (AArch64 Cortex-A72), using the toolchain in `output/host/`.
- **Configuration:** `make <board>_defconfig` loads a config, `make menuconfig` edits it, `make savedefconfig` saves it. The `local-lts` branch tracks my changes.
- **SD card image layout:** an MBR table with two partitions: a **FAT32 boot partition** (firmware, `config.txt`, `cmdline.txt`, kernel, Device Trees) and an **ext4 rootfs partition** (the Linux system mounted as `/`).
- **Inspecting an image without hardware:** `fdisk -l` gives the partitions in sectors; × 512 gives the bytes needed by `mount -o loop,offset=…,sizelimit=…`.
- **Flashing:** done on the host with `dd` (or balenaEtcher), never from the container. Always check the disk number first.
- **Serial console:** the Pi is controlled through UART at 115200 bauds with picocom (TX ↔ RX crossed, common GND). Power the Pi last to see all the boot logs.
- **Boot chain:** vanilla: ROM → `start4.elf` → Linux kernel → rootfs. With U-Boot: ROM → `start4.elf` → **U-Boot** → Linux kernel → rootfs. U-Boot adds control over the boot process.
- **Customizing the system:** hostname, banner and root password are set in _System configuration_; the boot files are controlled by `config_4_64bit.txt` and `genimage.cfg.in`.
- **Host tool compatibility:** Buildroot checks the host tools before building; the Rust `install` of Ubuntu 26.04 had to be replaced by the GNU version.
