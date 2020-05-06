# TPM 2.0 in U-Boot on Raspberry Pi 4

This guide shows how to use the TPM 2.0 in the `U-Boot` bootloader prior to loading
the Linux kernel on Raspberry Pi 4. In our case, we want U-Boot and the Linux
Kernel to be 64 bit because why not. :)

DISCLAIMER: since U-Boot does not ship a hardware SPI driver for Raspberry, both
U-Boot and Linux use their built-in soft-spi driver to communicate to the TPM.

## No Secure Boot on Raspberry Pi

Secure boot on the Raspberry Pi is not possible. That is because the first-stage
bootloader on the raspberry (`bootcode.bin` and `start.elf`) is closed source.
For secure boot, you need a so-called *Root of Trust* in the first-stage
bootloader, and we do not have that.

Actually, there is an [open-source first-stage
bootloader](https://github.com/christinaa/rpi-open-firmware) implemented mostly
via reverse-engineering. Unfortunately, this project has its limitations and is
currently on an indefinite hold.

## Pre-boot TPM

We want to be able to use the TPM prior to booting the Linux kernel. To do that,
we need to add a second-stage bootloader (`u-boot` in our case) with TPM
support.

The boot chain on the Raspberry Pi:

```
+-----------------+                                           +------------------------+
|    first-stage  |                                           |        Raspbian        |
|    bootloader   |------------------------------------------\|      Linux Kernel      |
|                 |------------------------------------------/|                        |
| (closed-source) |                                           | (built-in TPM support) |
+-----------------+                                           +------------------------+
```

What we want to archive

```
+-----------------+        +-------------------------+        +------------------------+
|    first-stage  |        | second-stage bootloader |        |        Raspbian        |
|    bootloader   |-------\|          U-Boot         |-------\|      Linux Kernel      |
|                 |-------/|                         |-------/|                        |
| (closed-source) |        | (built-in TPM support)  |        | (built-in TPM support) |
+-----------------+        +-------------------------+        +------------------------+
```

## Preparing your Raspberry Pi

Get the headless Raspbian image.

```bash
wget -O raspian_latest.zip https://downloads.raspberrypi.org/raspbian_lite_latest
unzip raspbian_lastest.zip
```

Check the character device name of your SD card with `lsblk` if needed. Plug
your SD card in, unmount its partition if necessary and flash the Raspbian image
onto the card:

```bash
sudo dd if=2020-02-13-raspbian-buster-lite.img of=/dev/mmcblk0 bs=4M status=progress conv=fsync
```

Done. But don't unplug your SD just yet. There should be two partitions on the
SD card now, `boot` and `rootfs`. Make sure they are auto-mounted now (`lsblk`).
You might need to re-plug your card.

## Getting a TPM

There are various options. I chose the [Lets Trust TPM](https://buyzero.de/collections/andere-platinen/products/letstrust-hardware-tpm-trusted-platform-module?variant=33890452626). It's cheap and it's for Raspberry Pi. (Seriously, who doesn't hate jumper wires?)

## Getting a Cross-Compiler

We are on an `ARMv8-A` processor and want to compile 64 bit software, i.e.
the architecture we want to build for is `aarch64`/`arm64`.

This means, we need the `aarch64-linux-gnu` toolchain. You can build it yourself
and add it to your `$PATH` or [install
it](https://www.archlinux.org/packages/community/x86_64/aarch64-linux-gnu-gcc/)
with your superiour Linux distro's package manager.

Check if your toolchain is working:

``` bash
aarch64-linux-gnu-gcc --version
```

## Getting a 64 Bit Kernel

You have two options:
* Option A) Build the kernel yourself
* Option B) Update your 32 bit kernel to 64 bit

In any case we need to tell our bootloader to load the kernel in 64 bit mode.
We simply add the following line to `config.txt` on the boot partition.

```ini
arm_64bit=1
```

### Option A) Building the 64 Bit Kernel

Build the kernel on your developer machine:

```bash
git clone https://github.com/raspberrypi/linux
cd linux
make O=result ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- bcm2711_defconfig
make O=result ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu-
```

That's it! Make sure the root partition on your SD card is mounted (for me:
`/run/media/johannes/rootfs`). To copy the newly built kernel, its modules,
device tree and overlays:

```bash
sudo env PATH=$PATH make O=result ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- INSTALL_MOD_PATH=/run/media/johannes/rootfs modules_install

KERNEL=kernel8.img
sudo cp /run/media/johannes/boot/$KERNEL /run/media/johannes/boot/$KERNEL.bak
sudo cp result/arch/arm64/boot/Image /run/media/johannes/boot/$KERNEL
sudo cp result/arch/arm64/boot/dts/broadcom/*.dtb /run/media/johannes/boot/
sudo cp result/arch/arm64/boot/dts/overlays/*.dtb* /run/media/johannes/boot/overlays/
```

Since we still need to add U-Boot, don't unplug your card, yet.

### Option B) Updating your 32 Bit Kernel to 64 Bit

Alternatively, you can instruct your Raspberry to perform a kernel update and reboot.

```
sudo rpi-update
sudo reboot
```

After the reboot, shut your Raspbian off and plug in the SD card to your PC.

## Building U-Boot

The Raspberry Pi talks to the TPM via SPI. Now here's the catch:

U-Boot does not ship an SPI driver for our Raspberry's Chip. This means we
cannot use the SPI hardware controller with U-Boot. However, we can use a
software SPI driver which uses GPIO to *bit bang* the data to the TPM. Luckily
for us, U-Boot provides a ready-to-use driver exactly for that.

Yes, there is a second catch: this driver does not support the SPI mode 0
(CPOL=0/CPHA=0) which we need. We can work around this, but I'll come to that
later

### Setting up and Configuring

Clone the repository and create a the Raspberry Pi 4 default configuration.

``` bash
git clone https://gitlab.denx.de/u-boot/u-boot.git
cd u-boot
make -j$(nproc) CROSS_COMPILE=aarch64-linux-gnu- rpi_4_defconfig
```

The configuration is saved in `.config`. Now, we need to change some things.
Similar to the Linux kernel, there is an interactive tool for configuring.

``` bash
make menuconfig
```

Now the menu is quite convoluted and some options only turn up after other
options have been enabled. Navigate through the menu and enable (`y`) in this
order:

* Device Drivers
  * (y) SPI Support
    * (y) Enable Driver Model for SPI drivers     (`DM_SPI`)
* Library routines
  * Security Support
    * (y) Trusted Platform Module (TPM) support   (`TPM`)
* Device Drivers
  * TPM support
    * (y) TPMv2.x support                         (`TPM_V2`)
    * (y) Enable support for TPMv2.x SPI chips    (`TPM2_TIS_SPI`)
* Command line interface
  * Security commands
    * (y) Enable the 'tpm' command                (`CMD_TPM`)

Save to `.config` and exit the menu.

Note: in this menu, you can also enable logging for troubleshooting.

### Patching and Building

Remember how I said that we need to work around the problem of our SPI driver
not being able to operate at SPI mode 0? Now is the time.

Secondly, there is a compile-time bug. Just apply these two patches. (If the
patches do not apply, call `git checkout 7dbafe06` first.)

```bash
git apply /path/to/dm-spi-fix-CPHA-and-implement-CPOL-for-soft-spi.diff
git apply /path/to/fix_compile_time_bug.diff
```

Now you can build U-Boot.

```bash
make -j$(nproc) CROSS_COMPILE=aarch64-linux-gnu- all
```

### Creating the Boot Script

The most important result is `u-boot.bin`, our second stage bootloader. However,
to tell what to do (which kernel to load etc.), we need a second script-like
file.

Copy this into a file named `boot.scr`. Note that we specify the kernel which is
to be booted by U-Boot later (`kernel8.img`).

```
fdt addr ${fdt_addr} && fdt get value bootargs /chosen bootargs
fatload mmc 0:1 ${kernel_addr_r} kernel8.img
booti ${kernel_addr_r} - ${fdt_addr}
```

This boot script needs to be converted into a binary format which U-Boot can
parse. We call that file `boot.scr.uimg`.

```bash
./tools/mkimage -A arm64 -T script -C none -n "Boot script" -d boot.scr boot.scr.uimg
```

### TPM Device Overlay

The last thing U-Boot needs is a description of the hardware (e.g. which pins is
the TPM connected to, which drivers to load etc.). All this information is
contained in the **device tree** (`bcm2711-rpi-4-b.dtb` on the Raspberry Pi). To
make changes to the device tree, we create a **device tree overlay**.

Create a file named `tpm-soft-spi.dts` and copy the following into it.

```dts
/*
 * Device Tree overlay for the Infineon SLB9670 Trusted Platform Module add-on
 * boards, which can be used as a secure key storage and hwrng.
 * available as "Iridium SLB9670" by Infineon and "LetsTrust TPM" by pi3g.
 */

/dts-v1/;
/plugin/;

/ {
	compatible = "brcm,bcm2835", "brcm,bcm2708", "brcm,bcm2709";

	fragment@0 {
		target = <&spi0>;
		__overlay__ {
			compatible = "spi-gpio";
			pinctrl-names = "default";
			pinctrl-0 = <&spi0_gpio7>;
			gpio-sck = <&gpio 11 0>;
			gpio-mosi = <&gpio 10 0>;
			gpio-miso = <&gpio 9 0>;
			cs-gpios = <&gpio 7 0>;
			spi-delay-us = <0>;
			#address-cells = <1>;
			#size-cells = <0>;
			status = "okay";

			/* for kernel driver */
			sck-gpios = <&gpio 11 0>;
			mosi-gpios = <&gpio 10 0>;
			miso-gpios = <&gpio 9 0>;
			num-chipselects = <1>;

			slb9670: slb9670@0 {
				compatible = "tis,tpm2-spi", "infineon,slb9670";
				reg = <0>;
				gpio-reset = <&gpio 24 1>;
				#address-cells = <1>;
				#size-cells = <0>;
				status = "okay";

				/* for kernel driver */
				spi-max-frequency = <1000000>;
			};
		};
	};

	fragment@1 {
		target = <&spi0_gpio7>;
		__overlay__ {
			brcm,pins = <7 8 9 10 11 24>;
			brcm,function = <0>;
		};
	};

	fragment@2 {
		target = <&spidev1>;
		__overlay__ {
			status = "disabled";
		};
	};
};

```

Again, this file needs to be compiled into binary format using the device tree
compiler `dtc`.

```bash
dtc -O dtb -b 0 -@ tpm-soft-spi.dts -o tpm-soft-spi.dtbo
```

### Adding U-Boot to the Boot Chain

Now is the time to copy all U-Boot-related files onto the SD card. For me, the
SD card's `boot` partition is mounted to `/run/media/johannes/boot/`

```bash
cp u-boot.bin /run/media/johannes/boot/
cp boot.scr.uimg /run/media/johannes/boot/
cp tpm-soft-spi.dtbo /run/media/johannes/boot/overlays
```

Additionally, we need to instruct the Raspberry's first-stage bootloader to use
our TPM device tree overlay and load U-Boot instead of the Linux kernel. Make
sure the following lines are in `config.txt`:

```ini
arm_64bit=1

dtparam=spi=on
dtoverlay=tpm-soft-spi

# if you want to use the serial console
enable_uart=1

kernel=u-boot.bin
```

Unmount the SD card and you are good to go!

```bash
sudo umount /run/media/johannes/boot
sudo umount /run/media/johannes/rootfs
```

# Testing

Connect your serial-to-USB converter to the Raspberry and open the terminal.
Boot your board and once the U-Boot bootloader starts, interrupt to enter
commands:

```
tpm2 init
tpm2 startup TPM2_SU_CLEAR
tpm2 get_capability 0x6 0x106 0x200 2
```

For an Infineon TPM you should get `0x534c4239` and `0x36373000` which is hex
for `SLB9670`. Congrats!

After calling `boot`, your Linux kernel should boot. Here, you can access your
TPM via `/dev/tpm0`.
