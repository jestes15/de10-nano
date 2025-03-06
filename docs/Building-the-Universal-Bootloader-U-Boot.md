<p align="right"><sup><a href="Building-Embedded-Linux.md">Back</a> | <a href="Building-the-Kernel.md">Next</a> | </sup><a href="../README.md#getting-started"><sup>Contents</sup></a>
<br/>
<sup>Building Embedded Linux - Full Custom</sup></p>

# Building the Universal Bootloader (U-Boot)

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->

- [Summary](#summary)
- [Steps](#steps)
  - [Getting the sources](#getting-the-sources)
  - [Configuring](#configuring)
	- [Creating a new branch](#creating-a-new-branch)
	- [Add I2C Buses to DTS File](#add-i2c-buses-to-dts-file)
	- [Configure U-Boot to flash FPGA automatically at boot time](#configure-u-boot-to-flash-fpga-automatically-at-boot-time)
	- [Asssign a permanent mac address to the ethernet device](#asssign-a-permanent-mac-address-to-the-ethernet-device)
  - [Building](#building)
- [References](#references)
- [Appendix](#appendix)
  - [Setting the mac address at boot time](#setting-the-mac-address-at-boot-time)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

## Summary

U-Boot is a universal bootloader. Truly. It is used in almost every known embedded device out there and the DE10-Nano is no exception. Here we will build a U-Boot image to be used for the DE10-Nano.

This step will also generate the Secondary Program Loader (SPL) along with the bootloader. To understand how all the pieces fit together, refer to this table originally shared on [Stack Overflow](https://stackoverflow.com/questions/31244862/what-is-the-use-of-spl-secondary-program-loader/31252989). We will generate both steps 2 and 3.

```
+--------+----------------+----------------+----------+
| Boot   | Terminology #1 | Terminology #2 | Actual   |
| stage  |                |                | program  |
| number |                |                | name     |
+--------+----------------+----------------+----------+
| 1      |  Primary       |  -             | ROM code |
|        |  Program       |                |          |
|        |  Loader        |                |          |
|        |                |                |          |
| 2      |  Secondary     |  1st stage     | u-boot   |
|        |  Program       |  bootloader    | SPL      |
|        |  Loader (SPL)  |                |          |
|        |                |                |          |
| 3      |  -             |  2nd stage     | u-boot   |
|        |                |  bootloader    |          |
|        |                |                |          |
| 4      |  -             |  -             | kernel   |
|        |                |                |          |
+--------+----------------+----------------+----------+
```

## Steps

### Getting the sources

There are two source repositories for U-Boot - the official [U-Boot repo](https://github.com/u-boot/u-boot) and the [altera fork](https://github.com/altera-opensource/u-boot-socfpga) of the U-Boot repo. You can use either of them, but the ALtera fork has code specific to SoC FPGAs, but is mostly the same as the official repository. For this guide, we will be using the Altera fork and modifying it further.

Clone the repository:

```bash
cd $DEWD
git clone https://github.com/altera-opensource/u-boot-socfpga.git
```

List all the tags and select a release that you want to use. For this guide, I used the latest stable release that worked `socfpga_v2024.07`:

```bash
cd $DEWD/u-boot-socfpga

# List all available tags.
git tag

# Checkout the desired release.
git checkout socfpga_v2024.07
```

### Configuring

Here we will make a few changes to the source code to make it work more nicely with our DE10-Nano, viz.,

- Enable 4 I2C busses, one for the ADXL345 and one for the HDMI controller
- Configure U-Boot to flash FPGA automatically at boot time.
- Assign a permanent mac address to the ethernet device.

These steps are optional i.e. you can skip them and your bootloader will work great and you will only lose these 2 capabilities. If you prefer to get something up and running asap, then you can skip directly to Part 2 and come back to it later when you feel you need it.

#### Creating a new branch

We want to keep our changes separate from the branch synced with the repo. So let's create a new branch:

```bash
git checkout -b socfpga_v2024.07_custom_configuration
```

#### Add I2C Buses to DTS File

A DTS File, or a Devicetree Control file, provides run-time configuration of U-Boot via a flattened devcietree. This feature makes it possible for a single binary to support multiple development boards.

We will edit the following file and add the required information:

```bash
cd $DEWD/u-boot-socfpga
nano arch/arm/dts/socfpga_cyclone5_de10_nano.dts
```

Near the end of the file, look for the lines:

```bash
&portc {
	bank-name = "portc";
};

&mmc0 {
	status = "okay";
	bootph-all;
};
```

Between portc and mmc0 add the following:

```bash
&i2c0 {
	status = "okay";
	clock-frequency = <100000>;

	adxl345: adxl345@53 {
		compatible = "adi,adxl345";
		reg = <0x53>;

		interrupt-parent = <&portc>;
		interrupts = <3 2>;
	};
};

&i2c1 {
	status = "okay";
	clock-frequency = <100000>;
};

&i2c2 {
	status = "okay";
	clock-frequency = <100000>;
};

&i2c3 {
	status = "okay";
	clock-frequency = <100000>;
};
```

By adding these to the Devicetree file we enable all four I2C buses and set their clock frequency. We also add a device, the ADXL345, and specify its I2C address and interrupt port.

After making these changes, you can save them via:

```bash
git add .
git commit -m "Added mac address"
```


#### Configure U-Boot to flash FPGA automatically at boot time

One of the features of U-Boot is the ability to flash the FPGA with a binary design at boot time. If you have a design that you want running on the FPGA every time you turn on the device, this is a very useful feature. How to use this is explained in another section, but for now, we need to add some commands that will do this automatically.

We will edit the following file for this:

```bash
cd $DEWD/u-boot
nano include/configs/socfpga_de10_nano.h
```

Towards the end of the file, look for the following lines:

```bash
/* Memory configurations */
#define PHYS_SDRAM_1_SIZE		0x40000000	/* 1GiB */

/* The rest of the configuration is shared */
#include <configs/socfpga_common.h>
```

We will add configuration settings to load the FPGA at boot. So modify it to look like this:

```bash
/* Memory configurations */
#define PHYS_SDRAM_1_SIZE		0x40000000	/* 1GiB */

#define CONFIG_BOOTFILE		"zImage"
#define CONFIG_BOOTARGS		"console=ttyS0," __stringify(CONFIG_BAUDRATE)

#ifndef CFG_EXTRA_ENV_SETTINGS
#define CFG_EXTRA_ENV_SETTINGS \
	"verify=n\0" \
	"bootimage=" CONFIG_BOOTFILE "\0" \
	"fdt_addr=100\0" \
	"fdtfile=" CONFIG_DEFAULT_FDT_FILE "\0" \
	"bootm_size=0xa000000\0" \
	"kernel_addr_r="__stringify(CONFIG_SYS_LOAD_ADDR)"\0" \
	"fdt_addr_r=0x02000000\0" \
	"scriptaddr=0x02100000\0" \
	"scriptfile=u-boot.scr\0" \
	"fpga_file=soc_system.rbf\0" \
    "fatscript=" \
		"if test -e mmc 0:1 ${scriptfile}; then " \
			"echo --- Found ${scriptfile} ---; " \
			"load mmc 0:1 ${scriptaddr} ${scriptfile}; " \
        	"source ${scriptaddr}; " \
		"fi;\0" \
	"pxefile_addr_r=0x02200000\0" \
	"ramdisk_addr_r=0x02300000\0" \
    "socfpga_legacy_reset_compat=1\0" \
	"prog_core=if load mmc 0:1 ${loadaddr} fit_spl_fpga.itb;" \
		"then fpga loadmk 0 ${loadaddr}:fpga-core-1; fi\0" \
	"fpga_cfg=" \
		"env exists fpga_file || setenv fpga_file ${board}.rbf; " \
		"if test -e mmc 0:1 ${fpga_file}; then " \
			"load mmc 0:1 ${kernel_addr_r} ${fpga_file}; " \
			"fpga load 0 ${kernel_addr_r} ${filesize}; " \
			"bridge enable; " \
		"fi;\0" \
	BOOTENV

#endif

#define CONFIG_BOOTCOMMAND "run fatscript; run fpga_cfg; run distro_bootcmd"

/* The rest of the configuration is shared */
#include <configs/socfpga_common.h>

```

Quick explanation of what is happening here. `CONFIG_BOOTFILE` and `CONFIG_BOOTARGS` sets up default arguments that can be overided later on. If you look at `include/configs/socfpga_common.h` you will see a lot of the same settings in there as you do here. Since I am defining CFG_EXTRA_ENV_SETTINGS here and no in `include/configs/socfpga_common.h` that code is never used and must be copied over to still be used. I go on to define CONFIG_BOOTCOMMAND to call my custom scripts and then finally distro_bootcmd.

I added these to do two things:

1.  In the script that is named `fatscript` we check to see if the script file, defined in `scriptfile` a few lines above, exists in the fat partition of the MMC drive. If this file is found, we load it onto `scriptaddr` which is defined as `0x02100000`.

    Having the check for `u-boot.scr` is helpful to override the default sequence of commands by creating a boot script image file `u-boot.scr` and saving it in the fat partition. This way we don't have to compile u-boot and burn it to the sd card every time we need some custom configuration. This may come in handy in the future.

1.  Next in the boot process we check for if `fpga_file` exists on the fat partition, if this file does not exist we assume the file to be named ${board}.rbf and proceed. We try to load the rbf file from the fat partition to `kernel_addr_r`. We then load the rbf file from `kernel_addr_r` to the FPGA.

    Presumably,  `0x700000` is the size of the binary in bytes. But where does `0x700000` or `7MB` come from? Well, if we look in the [Cyclone V Device Data Sheet](https://www.intel.com/content/dam/www/programmable/us/en/pdfs/literature/hb/cyclone-v/cv_51002.pdf) on page 78, we see a table showing the size of `.rbf` configuration files in bits. Here is a screenshot:

    ![](images/uboot_rbf_size.png)

    The de10-nano uses a Cyclone V 5CSEBA6U23I7 and I've highlighted that above. So converting that from bits to MB, we get about `6.68MB` which is close to the `7MB` above.

After making the changes, save the file and exit. Then commit it to the branch.

```bash
git add .
git commit -m "Load FPGA on boot."
```

#### Asssign a permanent mac address to the ethernet device

By default, the ethernet device on the DE10-Nano isn't assigned a Mac address. So it assigns a random one every time you reboot the device. Which in turn leads to a different IP address every time, which makes sshing into it a bit tedious. You could run the commands to assign the mac address at boot time as shown in the appendix, but then you'll have to do that every time you make changes to the SD Card. By adding it to the source code, you avoid both these problems.

Let's generate a random mac address first. You can get it online if you like, but U-Boot already ships with one out of the box. Let's use that:

```bash
cd $DEWD/u-boot

# Compile the mac address generator.
make -C tools gen_eth_addr

# Run it!
tools/gen_eth_addr
```

You should get an address that looks like `56:6b:20:e9:4a:47`. Copy this and save it somewhere.

Now let's assign this to our device. To do this, open the following file in a text editor. I am using `nano` but you can use `vim`, `gedit` or any other editor:

```bash
cd $DEWD/u-boot
nano include/configs/socfpga_de10_nano.h
```

Scroll down to the section that has the following lines:

```bash
	"ramdisk_addr_r=0x02300000\0" \
    "socfpga_legacy_reset_compat=1\0" \
	"prog_core=if load mmc 0:1 ${loadaddr} fit_spl_fpga.itb;" \
		"then fpga loadmk 0 ${loadaddr}:fpga-core-1; fi\0" \
```

Modify this to add the U-Boot environment variable `ethaddr` as shown below. Don't forget the `\0` at the end as well as the `\` or it won't work.

```bash
	"ramdisk_addr_r=0x02300000\0" \
    "socfpga_legacy_reset_compat=1\0" \
	"ethaddr=56:6b:20:e9:4a:47\0" \
	"prog_core=if load mmc 0:1 ${loadaddr} fit_spl_fpga.itb;" \
		"then fpga loadmk 0 ${loadaddr}:fpga-core-1; fi\0" \
```

You can save the file and exit. We're done with this step. Let's commit our changes:

```bash
git add .
git commit -m "Added mac address"
```

> **Note**: To those familiar with version management, saving hard coded mac addresses, passwords and any other kind of sensitive data in a repository is a definite no-no. But since this is for hobby use and on my own personal router, and I only have one DE10-Nano board, it should be fine. If you are doing this for your company, you might not want to hard code the mac address in the header file, it is not scalable. Instead, you should follow the steps in the Appendix which show how to set the mac address in the U-Boot console on first boot. Also, you shouldn't rely on the random mac generator and you should actually buy some valid mac addresses. Refer to [this link](https://www.denx.de/wiki/view/DULG/WhereCanIGetAValidMACAddress) for more info.

### Building

U-Boot has a number of pre-built configurations in the `configs` folder. To view all the available ones for altera, run the following command:

```bash
cd $DEWD/u-boot
ls -l configs/socfpga*
```

We will be using `socfpga_de10_nano_defconfig`.

Prepare the default config:

```bash
make ARCH=arm socfpga_de10_nano_defconfig
```

The defaults should be fine. But should you choose to fine tune the config, you can run the following and update them:

```bash
make ARCH=arm menuconfig
```

Now we can build U-Boot. Run the following command:

```bash
make ARCH=arm -j 12
```

Once the compilation completes, it should have generated the file `u-boot-with-spl.sfp`. This is the bootloader combined with the secondary program loader (spl).

## References

[Official U-Boot repository](https://github.com/u-boot/u-boot) - The README has most of the instructions.

[Building embedded linux for the Terasic DE10-Nano](https://bitlog.it/20170820_building_embedded_linux_for_the_terasic_de10-nano.html) - This page is again a very useful reference.

## Appendix

### Setting the mac address at boot time

If you skipped the section on hardcoding the mac address, you can change it at boot time as well. When the device boots, you can interrupt autoboot and enter the following commands to set the mac address. Note that this particular environment variable can only be set once and once set, cannot be changed:

```bash
setenv ethaddr 56:6b:20:e9:4a:47
saveenv
```

<p align="right">Next | <b><a href="Building-the-Kernel.md">Building the Kernel</a></b>
<br/>
Back | <b><a href="Building-Embedded-Linux.md">The Basics</a></p>
</b><p align="center"><sup>Building Embedded Linux - Full Custom | </sup><a href="../README.md#building-embedded-linux---full-custom"><sup>Table of Contents</sup></a></p>
