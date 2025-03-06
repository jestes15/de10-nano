<p align="right"><sup><a href="Building-the-Universal-Bootloader-U-Boot.md">Back</a> | <a href="Building-the-Kernal-RootFS-Choose-One.md">Next</a> | </sup><a href="../README.md#getting-started"><sup>Contents</sup></a>
<br/>
<sup>Building Embedded Linux - Full Custom</sup></p>

# Building the Kernel

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->

  - [Summary](#summary)
  - [Steps](#steps)
    - [Library dependencies](#library-dependencies)
    - [Download the Kernel](#download-the-kernel)
    - [Configure the Kernel](#configure-the-kernel)
    - [Primer on Flashing FPGA](#primer-on-flashing-fpga)
    - [Kernel Options](#kernel-options)
      - [Do not append version](#do-not-append-version)
      - [Enable Overlay filesystem support](#enable-overlay-filesystem-support)
      - [Enable CONFIGFS](#enable-configfs)
      - [(Optional) Enable options for WIFI](#optional-enable-options-for-wifi)
    - [Build the Kernel image](#build-the-kernel-image)
- [References](#references)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

## Summary

Here we build a kernel for the DE10-Nano from scratch.

## Steps

### Library dependencies

The following packages are needed for compiling the kernel:

```bash
sudo apt-get install libncurses-dev flex bison openssl libssl-dev dkms libelf-dev libudev-dev libpci-dev libiberty-dev libmpc-dev libgmp3-dev autoconf bc
```

### Download the Kernel

Altera has their own fork of the kernel. You can clone the [altera linux repository](https://github.com/altera-opensource/linux-socfpga.git):

```bash
cd $DEWD
git clone https://github.com/altera-opensource/linux-socfpga.git
```

List the branches with `git branch -a` and checkout the one you want to use. We chose to use the latest available at the time of writing. What's the point of using an ancient kernel if you're going to all this trouble?

```bash
cd linux-socfpga

# View the list of available branches.
git branch -a

# Use the branch you prefer.
git checkout socfpga-6.6.37-lts
```

### Add the DE10-Nano Configuration files to the Kernel

There are a few changes to the kernel source that need to be made, such as adding a device tree file to the kernel for the DE10-Nano and changing the configuration to use the heartbeat LED.

First open the file `socfpga_cyclone5_de10_nano.dts` with the command:

```bash
nano arch/arm/boot/dts/intel/socfpga/socfpga_cyclone5_de10_nano.dts
```

Copy the following into the file and save it:

```
/*
 * Copyright Intel Corporation (C) 2017. All rights reserved.
 *
 * This program is free software; you can redistribute it and/or modify
 * it under the terms and conditions of the GNU General Public License,
 * version 2, as published by the Free Software Foundation.
 *
 * This program is distributed in the hope it will be useful, but WITHOUT
 * ANY WARRANTY; without even the implied warranty of MERCHANTABILITY or
 * FITNESS FOR A PARTICULAR PURPOSE.  See the GNU General Public License for
 * more details.
 *
 * You should have received a copy of the GNU General Public License along with
 * this program.  If not, see <http://www.gnu.org/licenses/>.
 */

#include "socfpga_cyclone5.dtsi"

/ {
	model = "Terasic DE10-Nano";
	compatible = "altr,socfpga-cyclone5", "altr,socfpga";

	chosen {
		bootargs = "earlyprintk";
		stdout-path = "serial0:115200n8";
	};

	memory {
		name = "memory";
		device_type = "memory";
		reg = <0x0 0x40000000>; /* 1GB */
	};

	aliases {
		ethernet0 = &gmac1;
	};

	regulator_3_3v: 3-3-v-regulator {
		compatible = "regulator-fixed";
		regulator-name = "3.3V";
		regulator-min-microvolt = <3300000>;
		regulator-max-microvolt = <3300000>;
	};

	leds {
		compatible = "gpio-leds";
		hps0 {
			label = "hps_led0";
			gpios = <&portb 24 0>;
			linux,default-trigger = "heartbeat";
		};
	};

	keys {
		compatible = "gpio-keys";
		hps0 {
			label = "hps_key0";
			gpios = <&portb 25 0>;
			linux,code = <63>;
			debounce-interval = <50>;
		};
	};
};

&gmac1 {
	status = "okay";
	phy-mode = "rgmii";

	txd0-skew-ps = <0>; /* -420ps */
	txd1-skew-ps = <0>; /* -420ps */
	txd2-skew-ps = <0>; /* -420ps */
	txd3-skew-ps = <0>; /* -420ps */
	rxd0-skew-ps = <420>; /* 0ps */
	rxd1-skew-ps = <420>; /* 0ps */
	rxd2-skew-ps = <420>; /* 0ps */
	rxd3-skew-ps = <420>; /* 0ps */
	txen-skew-ps = <0>; /* -420ps */
	txc-skew-ps = <1860>; /* 960ps */
	rxdv-skew-ps = <420>; /* 0ps */
	rxc-skew-ps = <1680>; /* 780ps */

	max-frame-size = <3800>;
};

&gpio0 {
	status = "okay";
};

&gpio1 {
	status = "okay";
};

&gpio2 {
	status = "okay";
};

&i2c0 {
	status = "okay";
	speed-mode = <0>;

	adxl345: adxl345@53 {
		compatible = "adi,adxl345";
		reg = <0x53>;

		interrupt-parent = <&portc>;
		interrupts = <3 2>;
	};
};

&i2c1 {
	status = "okay";
	speed-mode = <0>;
};

&i2c2 {
	status = "okay";
	speed-mode = <0>;
};

&i2c3 {
	status = "okay";
	speed-mode = <0>;
};

&mmc0 {
	vmmc-supply = <&regulator_3_3v>;
	vqmmc-supply = <&regulator_3_3v>;
	status = "okay";
};

&uart0 {
	status = "okay";
};

&uart1 {
        status = "okay";
};

&usb1 {
	status = "okay";
};

&fpga_bridge0 {
  status = "okay";
  bridge-enable = <1>;
};

&fpga_bridge1 {
  status = "okay";
  bridge-enable = <1>;
};

&fpga_bridge2 {
  status = "okay";
  bridge-enable = <1>;
};

&fpga_bridge3 {
  status = "okay";
  bridge-enable = <1>;
};
```

For the build system to see the dts file, the Makefile corresponding to it needs to be modified. Open the `Makefile` with the following command:

```bash
nano arch/arm/boot/dts/intel/socfpga/Makefile
```

Add the following to the file just below the dtb for the DE0-Nano:

```bash
socfpga_cyclone5_de10_nano_soc.dtb \
```

Lastly, add the following to the file `arch/arm/configs/socfpga_defconfig` file:

```bash
CONFIG_KEYBOARD_GPIO_POLLED=y
CONFIG_INPUT_POLLDEV=y
CONFIG_KEYBOARD_GPIO=y
CONFIG_INPUT_MISC=y
CONFIG_INPUT_ADXL34X=y
CONFIG_INPUT_ADXL34X_I2C=y
CONFIG_INPUT_ADXL34X_SPI=y
CONFIG_LEDS_TRIGGER_ONESHOT=y
CONFIG_LEDS_TRIGGER_HEARTBEAT=y
CONFIG_LEDS_TRIGGER_BACKLIGHT=y
CONFIG_LEDS_TRIGGER_GPIO=y
CONFIG_LEDS_TRIGGER_DEFAULT_ON=y
CONFIG_FONT_8x8=y
CONFIG_FONT_8x16=y
CONFIG_LOGO=y
CONFIG_LOGO_LINUX_MONO=y
CONFIG_LOGO_LINUX_VGA16=y
CONFIG_LOGO_LINUX_CLUT224=y
CONFIG_FONT_SUPPORT=y
CONFIG_FB_CMDLINE=y
CONFIG_FB_SIMPLE=y
CONFIG_FB_CFB_FILLRECT=y
CONFIG_FB_CFB_COPYAREA=y
CONFIG_FB_CFB_IMAGEBLIT=y
CONFIG_LOCALVERSION_AUTO=n
CONFIG_OVERLAY_FS=y
CONFIG_OVERLAY_FS_REDIRECT_DIR=y
CONFIG_OVERLAY_FS_REDIRECT_ALWAYS_FOLLOW=y
CONFIG_OVERLAY_FS_INDEX=y
CONFIG_OVERLAY_FS_NFS_EXPORT=y
CONFIG_OVERLAY_FS_METACOPY=y
CONFIG_OVERLAY_FS_DEBUG=y
```

### Configure the Kernel

Initialize the configuration for the DE10-Nano:

```bash
make ARCH=arm socfpga_defconfig
```

Now open the kernel configuration window:

```bash
make ARCH=arm menuconfig
```

There are a few adjustments we want to make. But before that, let's understand a little bit about how to flash the FPGA on your DE10-Nano.

### Primer on Flashing FPGA

There are a few ways to flash your FPGA design. The traditional way is to just connect to the USB blaster interface and just flash it away. However, with the ARM HPS on our SoC, we have a couple of other ways as well:

1. **Flash from U-Boot on boot** - U-Boot has the ability to flash the FPGA design using some built-in commands. We will visit how to do this when we're building U-Boot.
2. **Flash from Linux while running** - I think this is one of the big benefits of having an HPS. We can flash our hardware design directly from Linux without even rebooting the device.

To be able to do this, we need to enable the **Overlay filesystem support** and **Userspace-driven configuration filesystem** in the kernel. If you don't intend to flash your FPGA from linux then feel free to skip these.

> **NOTE** - If you don't need these 2 options, you can just use the mainstream linux source to build your kernel instead of the altera-opensource version. Just clone the repository at `github.com/torvalds/linux`.

### Build the Kernel image

Now we can finally build the kernel image. Use the following command to create a kernel image called `zImage`:

```bash
make ARCH=arm LOCALVERSION=zImage -j 12
```

If it makes any complaints about `bc` not found or `flex` not found, install that utility using `sudo apt install <library>`.

The kernel gets compiled in about 5-10 mins on my Virtualbox Debian, but YMMV.

Once the compilation is complete, you now have a compressed Linux kernel image.

# References

[Building embedded linux for the Terasic DE10-Nano](https://bitlog.it/20170820_building_embedded_linux_for_the_terasic_de10-nano.html) - A large part of this page has been taken from here.

[Stackoverflow - Cannot mount configfs](https://stackoverflow.com/questions/50877808/configfs-do-not-mount-device-tree-overlays) - This page explains why you cannot see the device tree overlay.

<p align="right">Next | <b><a href="Building-the-Kernal-RootFS-Choose-One.md">RootFS - Choose one</a></b>
<br/>
Back | <b><a href="Building-the-Universal-Bootloader-U-Boot.md">Building the Universal Bootloader (U-Boot)</a></p>
</b><p align="center"><sup>Building Embedded Linux - Full Custom | </sup><a href="../README.md#building-embedded-linux---full-custom"><sup>Table of Contents</sup></a></p>
