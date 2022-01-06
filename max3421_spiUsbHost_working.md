### Steps taken to get the MAX3421 working on RPi 0 W 1.1 as of 06/01/2022
##### Nb: some parts on this writeup are not directly related to the object of interest but may help getting some basics ok

### TODO's ( among way to many others .. )
- learn how to mod & configure a kernel + update it 'the right way' ( either on RPi itself or learn a cross-compiling setup )
- try connecting LEd's / stuff to the GPOUTx of the Max3421 to debug driving its GPOUTx pins ( once in overlay & later using driver ) using cli/python/nodejs/C/..
- try uncommenting the '__overrides__' in the DT overlay to allow overriding the DT overlay params via 'dtparam's in /boot/config.txt
- try modifying the DT overlay to support different SPI & CE pins
- try allowing to pass 'MAX_GPX' & 'MAX_RESET' pins as well ( instead of connecting the latter to he RPi's 3.3V & not connecting the former at all ?)
- try impementing a way to receive GPINx 'evts' & getting those either 'globally' ( R 'old' pengpod hw btns devs ) or via cli/python/nodejs/C/..
- try connecting USB peripherals ( USB KEY tested ok, USB kbd/mouse/gamepad/HID/AirBar/.. 'whatever' ? :) )
- try mounting the USB KEY & accessing its file ( done )
- try mixing the max3421 support with USB Gadget ( mass_storage or better, Libcomposite ) & getting a 'backing store' using an USB KEY instead of .img on SD
- instad of starting with the current 'advanced' design on EasyEDA, try a V0.1a breakout board for the RPI to 'combo' with Pimoroni's 'pirateAudio' one ? :)
- =>thnk: usb port | thin connector, toggle for 5V/3.3V/self powered USB peripheral(s)

### Other TODO
- A 'clear enough' writeup is coming, once I'm sure to validate most of the above features for it ( aka, after posting the current POC & asking question to experts on the RPi forums ;p )
- Also, remember to ask why adding no .dtbo nor dtparams are provided for the max3421 even if it has a driver ( .c ) & even a seemingly-closely-related .yaml stuff ? ( max3421/0-ucd )

#### 1: prepare SD card
Ex: using 'Raspberry Pi imager'

#### 2: enable the basics

```
# go within the SD /boot partition
$ cd /Volumes/boot/

# -- activate ssh -- 
$ touch ssh

# -- wifi --
# add wifi network infos
cat << EOF >> ./wpa_supplicant.conf
country=FR # 2-digit county code
ctrl_interface=DIR=/var/run/wpa_supplicant GROUP=netdev # RPiOS, Raspbian Stretch/Buster ( no need for Raspbian Jessie or older
network={
    ssid="<your_ssid>"
    psk="<your_psk>"
    key_mgmt=WPA-PSK
}
EOF

# add support for USB Gadget & force-set dw2 role as peripheral ( to enable 'Slave' mode for both RPi0 & A+ version
# R: 'dr_mode' is visible in https://github.com/raspberrypi/firmware/blob/master/boot/overlays/README
$ echo "dtoverlay=dwc2,dr_mode=peripheral" | sudo tee -a /boot/config.txt
# have the g_serial module loaded
#echo -n " modules-load=dwc2,g_serial" | sudo tee -a /boot/cmdline.txt
#/bin/echo -n " modules-load=dwc2,g_serial" >> ./cmdline.txt 
# do it by hand for now ;/ ..

# -- R: for the "g_serial" to work, we need to have a tty tied to it --
# from: https://learn.adafruit.com/turning-your-raspberry-pi-zero-into-a-usb-gadget/serial-gadget
$ sudo systemctl enable getty@ttyGS0.service
# says: "Created symlink /etc/systemd/system/getty.target.wants/getty@ttyGS0.service → /lib/systemd/system/getty@.service."
# So, I guess we could pre-copy stuff there ? ( the symlink )
$ sudo systemctl is-active getty@ttyGS0.service # verify it actually runnning
# says: "inactive", so trying after & "sudo reboot" ..
# ok after reboot, we can "screen /dev/tty.usbmodem14401 115200"
# possible way to edit a file priorhand to not have to use the above cli ? ( or possibly unrelated yet interesting ):
# https://www.rogerirwin.co.nz/open-source/enabling-a-serial-port-console/
```

#### 3: kernel stuff, part 1 ( that is, testing a 'dumb' driver & the related cmds & expected behavior )
```
-- TRY 1, using a 'quick tutorial' found on RaspberyPi Stack Exchange, to avoid modding the whole kernel to try adding a module --
# --> https://raspberrypi.stackexchange.com/questions/39845/how-compile-a-loadable-kernel-module-without-recompiling-kernel
# getting up-to-date ( != rpi-update ! ): https://www.raspberrypi.com/documentation/computers/os.html#updating-and-upgrading-raspberry-pi-os
$ sudo apt update
$ sudo apt full-upgrade -y
E: Failed to fetch http://ftp.igh.cnrs.fr/pub/os/linux/raspbian/raspbian/pool/main/t/tzdata/tzdata_2021a-1+deb11u2_all.deb  Connection timed out [IP: 193.50.6.155 80]
E: Unable to fetch some archives, maybe run apt-get update or try with --fix-missing?
$ sudo apt full-upgrade --fix-missing -y

# just to know:
$ uname -r
# 5.10.63+

# getting the kernel headers ( watchout for versions mismatch according to ? )
$ sudo apt install raspberrypi-kernel-headers
# writing a quick test module to test if we can compile against the headers just downloaded
$ mkdir /home/pi/hellotef
$ cd !*

# create the .C module file
$ cat << EOF >> ./hellotef.c
#include <linux/init.h>  
#include <linux/kernel.h> 
#include <linux/module.h>

MODULE_LICENSE("GPL");
MODULE_DESCRIPTION("Do-nothing hellotef test driver");
MODULE_VERSION("0.1");

static int __init hello_init(void){
   /*pr_alert("Hello, Tef.\n");*/
   printk(KERN_INFO "Hello, Tef.\n");
   return 0;
}

static void __exit hello_exit(void){
  /*pr_alert("Goodbye, Tef.\n");*/
   printk(KERN_INFO "Goodbye,Tef.\n");
}

module_init(hello_init);
module_exit(hello_exit);
EOF

# create the related Makefile
# /!\ it said -bash: sell: command not found", so take care if it replaces the stuff within '$()'
# indeed, all the 'M=$(pwd)' ahave been replaced by 'M=/home/pi/hellotef'
# also, maybe try replacing the calls containing '$(shell uname -r)' as just '$(uname -r)'
# also, watchout: use TABS instead of 4 SPACES if getting a Makefile error
$ cat << EOF >> ./Makefile
obj-m+=hellotef.o

all:
    make -C /lib/modules/$(shell uname -r)/build M=$(pwd) modules

clean:
    make -C /lib/modules/$(shell uname -r)/build M=$(pwd) clean

modules_install: all
    $(MAKE) -C $(KERNEL_SRC) M=$(SRC) modules_install
    $(DEPMOD)
EOF
# Usage
# 'make' or 'make all': same as 'make -C /lib/modules/$(uname -r)/build M=$(pwd) modules'
# Test:
# 1) insert module 'sudo insmod hellotef.ko'
# 2) check dmges 'dmesg | grep hello '
# 1) remove module 'sudo rmmod hellotef'
# 3) check dmges 'dmesg | grep hello '
# Once module is working, install using 'sudo make modules_install'

# Nb: the above didn't seem to work for me ( at least, without using 'sudo' )
# this being said, using another simpler Makefile containing just 'obj-m:=hellotef.o'
# then cli: 'make -C /lib/modules/$(uname -r)/build M=$(pwd) modules'
# getting the follling logs with no errors
make -C /lib/modules/$(uname -r)/build M=$(pwd) modules
make: Entering directory '/usr/src/linux-headers-5.10.63+'
  CC [M]  /home/pi/hellotef/hellotef.o
  MODPOST /home/pi/hellotef/Module.symvers
  CC [M]  /home/pi/hellotef/hellotef.mod.o
  LD [M]  /home/pi/hellotef/hellotef.ko
make: Leaving directory '/usr/src/linux-headers-5.10.63+'
# and it also produced a lot of files:
Module.symvers  hellotef.ko  hellotef.mod  hellotef.mod.c  hellotef.mod.o  hellotef.o  modules.order

# just to check if the downloaded kernel headers version matches the one I have with 'uname -r'
# https://www.raspberrypi.com/documentation/computers/linux_kernel.html#version-identification
# Quote: "in a kernel source directory, run: 'head Makefile -n 3'"
# doing so for the directory seen above, I get:
# # SPDX-License-Identifier: GPL-2.0
#VERSION = 5
#PATCHLEVEL = 10
# also, getting the same for each /usr/scr/ linux-headers-5.10.63+/     linux-headers-5.10.63-v7+/  linux-headers-5.10.63-v7l+/

# back to our dir to make quick loading/unloading & demsg checks
$ cd /home/pi/hellotef

# test load
$ sudo insmod hellotef.ko
$ lsmod | grep hello
# hellotef               16384  0
$ dmesg | grep "[h|H]ello"
$ dmesg | grep "[t|T]ef"
#[ 6350.932736] hellotef: loading out-of-tree module taints kernel.
#[ 6350.933286] Hello, Tef.

# test unload
$ sudo rmmod hellotef
$ dmesg | grep "[t|T]ef"
#[ 6350.932736] hellotef: loading out-of-tree module taints kernel.
#[ 6350.933286] Hello, Tef.
#[ 6586.089636] Goodbye, Tef.

# test install, using the non-working Makefile -> nope
$ sudo make modules_install
make -C /lib/modules/5.10.63+/build M= modules
make[1]: Entering directory '/usr/src/linux-headers-5.10.63+'
make[2]: *** No rule to make target 'arch/arm/tools/syscall.tbl', needed by 'arch/arm/include/generated/uapi/asm/unistd-common.h'.  Stop.
make[1]: *** [arch/arm/Makefile:307: archheaders] Error 2
make[1]: Leaving directory '/usr/src/linux-headers-5.10.63+'
make: *** [Makefile:4: all] Error 2

# trying the 'alternative' version on the cli ? ..
$ make -C /lib/modules/$(uname -r)/build M=$(pwd) modules_install
#make: Entering directory '/usr/src/linux-headers-5.10.63+'
#mkdir: cannot create directory '/lib/modules/5.10.63+/extra': Permission denied -----> SO IT WAS ABOUT TO INSTALL IT IN "EXTRAS" DIR, NOT USB/HOST/..
#make: *** [Makefile:1740: _emodinst_] Error 1
#make: Leaving directory '/usr/src/linux-headers-5.10.63+'
# retrying using sudo ..
$ sudo !!
sudo make -C /lib/modules/$(uname -r)/build M=$(pwd) modules_install
make: Entering directory '/usr/src/linux-headers-5.10.63+'
  INSTALL /home/pi/hellotef/hellotef.ko
  DEPMOD  5.10.63+
Warning: modules_install: missing 'System.map' file. Skipping depmod.
make: Leaving directory '/usr/src/linux-headers-5.10.63+'
# --> no errors, so trying a 'modprobe' ..
$ modprobe hellotef
#modprobe: FATAL: Module hellotef not found in directory /lib/modules/5.10.63+
# --> so, maybe cuz it skipped 'depmod' it doesn't find it ? ..
$ depmod
#depmod: ERROR: openat(/lib/modules/5.10.63+, modules.dep.6327.385028.1641366274, 301, 644): Permission denied
#depmod: ERROR: openat(/lib/modules/5.10.63+, modules.dep.bin.6327.385028.1641366274, 301, 644): Permission denied
#depmod: ERROR: openat(/lib/modules/5.10.63+, modules.alias.6327.385028.1641366274, 301, 644): Permission denied
#depmod: ERROR: openat(/lib/modules/5.10.63+, modules.alias.bin.6327.385028.1641366274, 301, 644): Permission denied
#depmod: ERROR: openat(/lib/modules/5.10.63+, modules.softdep.6327.385028.1641366274, 301, 644): Permission denied
#depmod: ERROR: openat(/lib/modules/5.10.63+, modules.symbols.6327.385028.1641366274, 301, 644): Permission denied
#depmod: ERROR: openat(/lib/modules/5.10.63+, modules.symbols.bin.6327.385028.1641366274, 301, 644): Permission denied
#depmod: ERROR: openat(/lib/modules/5.10.63+, modules.builtin.bin.6327.385028.1641366274, 301, 644): Permission denied
#depmod: ERROR: openat(/lib/modules/5.10.63+, modules.builtin.alias.bin.6327.385028.1641366274, 301, 644): Permission denied
#depmod: ERROR: openat(/lib/modules/5.10.63+, modules.devname.6327.385028.1641366274, 301, 644): Permission denied
# retrying with 'sudo' now that I know which files could have been updated
$ sudo depmod
# no errors ..
# retying modprobe
$ modprobe hellotef
modprobe: ERROR: could not insert 'hellotef': Operation not permitted
# --> permission denied, yet file found ? ;)
$ sudo modprobe hellotef
lsmod | grep tef
#hellotef               16384  0
# --> yup
$ modinfo hellotef
filename:       /lib/modules/5.10.63+/extra/hellotef.ko ----> IS INDEED IN /extras
version:        0.1
description:    Do-nothing hellotef test driver
license:        GPL
srcversion:     DE8C19E4FD43A9A52DB11EB
depends:        
name:           hellotef
vermagic:       5.10.63+ mod_unload modversions ARMv6 p2v8



```

#### 4: kernel stuff, part 2 ( following ~same path as for our 'dumb' driver for our max3421-hcd driver + custom DT overlay )
```
# -- so, trying the same with a max3421 driver DL-ed from he RPI github repo which has the same kernel version
# R: https://github.com/raspberrypi/linux/blob/rpi-5.10.y/drivers/usb/host/max3421-hcd.c
$ mkdir /home/pi/max3421-hcd # added -hcd suffix to reflect the name of the 'official' module
$ cd !*

# Digging the above link one parent up
# https://github.com/raspberrypi/linux/blob/rpi-5.10.y/drivers/usb/host/Kconfig
config USB_MAX3421_HCD
	tristate "MAX3421 HCD (USB-over-SPI) support"
	depends on USB && SPI
	help
	  The Maxim MAX3421E chip supports standard USB 2.0-compliant
	  full-speed devices either in host or peripheral mode.  This
	  driver supports the host-mode of the MAX3421E only.

	  To compile this driver as a module, choose M here: the module will
	  be called max3421-hcd.

# And at the very end of this file: https://github.com/raspberrypi/linux/blob/rpi-5.10.y/drivers/usb/host/Makefile
obj-$(CONFIG_USB_MAX3421_HCD)	+= max3421-hcd.o

# At the following url: https://github.com/raspberrypi/linux/blob/rpi-5.10.y/Documentation/devicetree/bindings/usb/maxim%2Cmax3421.txt
Maxim Integrated SPI-based USB 2.0 host controller MAX3421E

Required properties:
 - compatible: Should be "maxim,max3421"
 - spi-max-frequency: maximum frequency for this device must not exceed 26 MHz.
 - reg: chip select number to which this device is connected.
 - maxim,vbus-en-pin: <GPOUTx ACTIVE_LEVEL>
   GPOUTx is the number (1-8) of the GPOUT pin of MAX3421E to drive Vbus.
   ACTIVE_LEVEL is 0 or 1.
 - interrupts: the interrupt line description for the interrupt controller.
   The driver configures MAX3421E for active low level triggered interrupts,
   configure your interrupt line accordingly.

Example:

	usb@0 {
		compatible = "maxim,max3421";
		reg = <0>;
		maxim,vbus-en-pin = <3 1>;
		spi-max-frequency = <26000000>;
		interrupt-parent = <&PIC>;
		interrupts = <42>;
	};
# And at the following, a -udc.yaml one ?: https://github.com/raspberrypi/linux/blob/rpi-5.10.y/Documentation/devicetree/bindings/usb/maxim%2Cmax3420-udc.yaml
# SPDX-License-Identifier: (GPL-2.0 OR BSD-2-Clause)
%YAML 1.2
---
$id: http://devicetree.org/schemas/usb/maxim,max3420-udc.yaml#
$schema: http://devicetree.org/meta-schemas/core.yaml#

title: MAXIM MAX3420/1 USB Peripheral Controller

maintainers:
  - Jassi Brar <jaswinder.singh@linaro.org>

description: |
  The controller provices USB2.0 compliant FullSpeed peripheral
  implementation over the SPI interface.
  Specifications about the part can be found at:
    http://datasheets.maximintegrated.com/en/ds/MAX3420E.pdf
properties:
  compatible:
    enum:
      - maxim,max3420-udc
      - maxim,max3421-udc

  reg:
    maxItems: 1

  interrupts:
    items:
      - description: usb irq from max3420
      - description: vbus detection irq
    minItems: 1
    maxItems: 2

  interrupt-names:
    items:
      - const: udc
      - const: vbus
    minItems: 1
    maxItems: 2

  spi-max-frequency:
    maximum: 26000000

required:
  - compatible
  - reg
  - interrupts
  - interrupt-names

additionalProperties: false

examples:
  - |
      #include <dt-bindings/gpio/gpio.h>
      #include <dt-bindings/interrupt-controller/irq.h>
      spi0 {
            #address-cells = <1>;
            #size-cells = <0>;
            udc@0 {
                  compatible = "maxim,max3420-udc";
                  reg = <0>;
                  interrupt-parent = <&gpio>;
                  interrupts = <0 IRQ_TYPE_EDGE_FALLING>, <10 IRQ_TYPE_EDGE_BOTH>;
                  interrupt-names = "udc", "vbus";
                  spi-max-frequency = <12500000>;
            };
      };

# So, since I can't find any trace of a max3420/1-ucd.c, DL-ing the max3421-hcd.c one:
$ wget https://raw.githubusercontent.com/raspberrypi/linux/rpi-5.10.y/drivers/usb/host/max3421-hcd.c
# R: at the very end of it, we can find
static const struct of_device_id max3421_of_match_table[] = {
	{ .compatible = "maxim,max3421", },
	{},
};
MODULE_DEVICE_TABLE(of, max3421_of_match_table);

static struct spi_driver max3421_driver = {
	.probe		= max3421_probe,
	.remove		= max3421_remove,
	.driver		= {
		.name	= "max3421-hcd",
		.of_match_table = of_match_ptr(max3421_of_match_table),
	},
};

module_spi_driver(max3421_driver);

MODULE_DESCRIPTION(DRIVER_DESC);
MODULE_AUTHOR("David Mosberger <davidm@egauge.net>");
MODULE_LICENSE("GPL");

# Also, it seems we could have GPOUTx control
# L1647
/*
 * Set the MAX3421E general-purpose output with number PIN_NUMBER to
 * VALUE (0 or 1).  PIN_NUMBER may be in the range from 1-8.  For
 * any other value, this function acts as a no-op.
 */
static void
max3421_gpout_set_value(struct usb_hcd *hcd, u8 pin_number, u8 value)

# also interestin L1802
# max3421_of_vbus_en_pin

# So, following nearly the same path that for our simplests possible driver:
$ cat << EOF >> ./Makefile
obj-m:= max3421-hcd.o
EOF
# building the module
$ make -C /lib/modules/$(uname -r)/build M=$(pwd) modules
# make: Entering directory '/usr/src/linux-headers-5.10.63+'
  CC [M]  /home/pi/max3421-hcd/max3421-hcd.o
  MODPOST /home/pi/max3421-hcd/Module.symvers
  CC [M]  /home/pi/max3421-hcd/max3421-hcd.mod.o
  LD [M]  /home/pi/max3421-hcd/max3421-hcd.ko
make: Leaving directory '/usr/src/linux-headers-5.10.63+'

# so, trying a quick insmod
$ sudo insmod max3421-hcd.ko
# --> no errors
$ lsmod | grep max342
#max3421_hcd            24576  0
$ dmesg | grep max342
#[ 3535.914249] max3421_hcd: loading out-of-tree module taints kernel.

# trying rmmod
$ sudo rmmod max3421_hcd
# --> no error nor msg on dmesg

# trying a module install ( I KNOW it won't be intall to /usb/host/ but /extra instead .. )
$ sudo make -C /lib/modules/$(uname -r)/build M=$(pwd) modules_install
# few log msgs, no errors
# make: Entering directory '/usr/src/linux-headers-5.10.63+'
  INSTALL /home/pi/max3421-hcd/max3421-hcd.ko
  DEPMOD  5.10.63+
Warning: modules_install: missing 'System.map' file. Skipping depmod.
make: Leaving directory '/usr/src/linux-headers-5.10.63+'

# trying a depmod & modinfo
$ sudo depmod
$ modinfo max3421-hcd
filename:       /lib/modules/5.10.63+/extra/max3421-hcd.ko
license:        GPL
author:         David Mosberger <davidm@egauge.net>
description:    MAX3421 USB Host-Controller Driver
srcversion:     A6905DA6B467013134BFD58
alias:          of:N*T*Cmaxim,max3421C*
alias:          of:N*T*Cmaxim,max3421
depends:        
name:           max3421_hcd
vermagic:       5.10.63+ mod_unload modversions ARMv6 p2v8

# trying a modprobe
$ sudo modprobe max3421-hcd
# no errors
$ lsmod | grep max3421
#max3421_hcd            24576  0

$ dmesg | grep max342
# [ 3535.914249] max3421_hcd: loading out-of-tree module taints kernel.

# just to make sure driver is installed, the aliases list has been updated and contains it:
$ cat /lib/modules/5.10.63+/modules.alias | grep max342
alias of:N*T*Cmaxim,max3421C* max3421_hcd
alias of:N*T*Cmaxim,max3421 max3421_hcd

# So, trying to use the following .dts BUT modifying it to fit ?
cat << EOF >> ./max3421-hcd-overlay.dts
/dts-v1/;
/plugin/;

/ {
    compatible = "brcm,bcm2835";
    /* Disable spidev for spi0.0 - release resource */
    fragment@0 {
        target = <&spi0>;
        __overlay__ {
            status = "okay";
            spidev@0{
                status = "disabled";
            };
        };
    };

    /* Set pins used (IRQ) */
    fragment@1 {
        target = <&gpio>;
        __overlay__ {
            max3421_pins: max3421_pins {
                brcm,pins = <25>;        //GPIO25
                brcm,function = <0>;    //Input
            };
        };
    };

    /* Create the MAX3421 node */
    fragment@2 {
        target = <&spi0>;
        __overlay__ {
            //avoid dtc warning 
            #address-cells = <1>;
            #size-cells = <0>;
            max3421: max3421@0 {
                reg = <0>;  //CS 0
                spi-max-frequency = <20000000>;
                compatible = "maxim,max3421";
                pinctrl-names = "default";
                pinctrl-0 = <&max3421_pins>;
                interrupt-parent = <&gpio>;
                interrupts = <25 0x2>;       //GPIO25, high-to-low
                maxim,vbus-en-pin = <1 1>;  //MAX GPOUT1, active high
                /*
                interrupts = <0 IRQ_TYPE_EDGE_FALLING>, <10 IRQ_TYPE_EDGE_BOTH>;
                interrupt-names = "udc", "vbus";
                */
            };
        };
    };

    __overrides__ {
        /* -- starting testing with overrides desactivated -- */
        /*spimaxfrequency = <&max3421>,"spi-max-frequency:0";*/
        /*interrupt = <&max3421_pins>,"brcm,pins:0",<&max3421>,"interrupts:0";*/
        /*vbus-en-pin = <&max3421>,"maxim,vbus-en-pin:0";*/
        /*vbus-en-level = <&max3421>,"maxim,vbus-en-pin:4";*/
    };
};
EOF



# R: 'old' example
---
usb@0 {
		compatible = "maxim,max3421";
		reg = <0>;
		maxim,vbus-en-pin = <3 1>;
		spi-max-frequency = <26000000>;
		interrupt-parent = <&PIC>;
		interrupts = <42>;
---

# R: 'new' example
---
#include <dt-bindings/gpio/gpio.h>
      #include <dt-bindings/interrupt-controller/irq.h>
      spi0 {
            #address-cells = <1>;
            #size-cells = <0>;
            udc@0 {
                  compatible = "maxim,max3420-udc";
                  reg = <0>;
                  interrupt-parent = <&gpio>;
                  interrupts = <0 IRQ_TYPE_EDGE_FALLING>, <10 IRQ_TYPE_EDGE_BOTH>;
                  interrupt-names = "udc", "vbus";
                  spi-max-frequency = <12500000>;
            };
      };
---

# Now, to build our .dtb file, we need the dtc tool
# as explained in https://www.raspberrypi.com/documentation/computers/configuration.html#part2.1
# WE COULD use: dtc -@ -Hepapr -I dts -O dtb -o 1st.dtbo 1st-overlay.dts
# ( -@ to allow unresolved references & -Hepapr to remove some clutter )
# The doc also says:  suitable compiler is also available in the kernel tree as scripts/dtc/dtc, built when the dtbs make target is used
# indeed, I can find one in :/usr/src/linux-headers-5.10.63+/scripts/dtc/dtc

# So, trying to use the one used globally:
$ dtc -@ -I dts -O dtb -o ./max3421-hcd.dtbo ./max3421-hcd-overlay.dts
# ./max3421-hcd-overlay.dts:11.21-13.15: Warning (unit_address_vs_reg): /fragment@0/__overlay__/spidev@0: node has a unit name, but no reg property

# On a 2nd ssh session, in another terminal tab, I did
$ cd /home/pi/max3421-hcd/
$ mkdir udevLogs
$ cd udevLogs/
$ udevadm monitor > ./udevlogs.txt

# "Dynamic Device Tree" to the rescue to avoid having to copy to /boot/overlays & reboot in order to check behavior ?
# see: https://www.raspberrypi.com/documentation/computers/configuration.html#part3.5
# Quote "changes are immediately reflected in /proc/device-tree and can cause modules to be loaded and platform devices to be created and destroyed."
# -- 'dtoverlay' command --
# 1) Unlike the config.txt equivalent, all parameters to an overlay must be included in the same command line - the dtparam command is only for parameters of the base DTB.
# 2) Command variants that change kernel state (adding and removing things) require root privilege, so you may need to prefix the command with sudo.
# 3) Only overlays and parameters applied at run-time can be unloaded - an overlay or parameter applied by the firmware becomes "baked in" such that it won’t be listed by dtoverlay and can’t be removed.
# 4) Removing an overlay will not cause a loaded module to be unloaded, but it may cause the reference count of some modules to drop to zero. Running rmmod -a twice will cause unused modules to be unloaded.
# 4) ==> the 'rmmod -a' cmd errored on the 1st try for me ( wrong -param ? no '-a' is supported ..  ), but 'sudo rmmod max3421_hcd' once removed it from lsmod

# So, trying to dynamically load our DT overlay & see if it causes the kernel driver to load
$ sudo dtoverlay ./max3421-hcd.dtbo
# -> no error ..
# listing loaded overlays this way
$ dtoverlay -l
#Overlays (in load order):
#0:  max3421-hcd
# checking lsmod to see if the module has loaded:
$ lsmod | grep max
# max3421_hcd            24576  0
# --> indeed, also, a sh*tload of logs:
$ cat ./udevLogs/udevlogs.txt
monitor will print the received events for:
UDEV - the event which udev sends out after rule processing
KERNEL - the kernel uevent

KERNEL[8470.767848] add      /devices/platform/soc/20204000.spi (platform)
KERNEL[8470.775406] add      /devices/virtual/devlink/platform:20101000.cprman--platform:20204000.spi (devlink)
KERNEL[8470.779785] add      /devices/virtual/devlink/platform:20007000.dma--platform:20204000.spi (devlink)
KERNEL[8470.780164] add      /devices/virtual/devlink/platform:20200000.gpio--platform:20204000.spi (devlink)
UDEV  [8470.878291] add      /devices/virtual/devlink/platform:20101000.cprman--platform:20204000.spi (devlink)
UDEV  [8470.884100] add      /devices/virtual/devlink/platform:20200000.gpio--platform:20204000.spi (devlink)
UDEV  [8470.891557] add      /devices/virtual/devlink/platform:20007000.dma--platform:20204000.spi (devlink)
KERNEL[8470.897106] add      /devices/platform/soc/20204000.spi/spi_master/spi0 (spi_master)
KERNEL[8470.899247] add      /devices/platform/soc/20204000.spi/spi_master/spi0/spi0.0 (spi)
KERNEL[8470.900976] add      /devices/virtual/devlink/platform:20200000.gpio--spi:spi0.0 (devlink)
UDEV  [8470.906974] add      /devices/virtual/devlink/platform:20200000.gpio--spi:spi0.0 (devlink)
KERNEL[8470.921082] add      /devices/platform/soc/20204000.spi/spi_master/spi0/spi0.0/usb1 (usb)
KERNEL[8470.923569] add      /devices/platform/soc/20204000.spi/spi_master/spi0/spi0.0/usb1/1-0:1.0 (usb)
KERNEL[8470.926134] bind     /devices/platform/soc/20204000.spi/spi_master/spi0/spi0.0/usb1/1-0:1.0 (usb)
KERNEL[8470.928297] bind     /devices/platform/soc/20204000.spi/spi_master/spi0/spi0.0/usb1 (usb)
KERNEL[8470.931797] remove   /devices/virtual/devlink/platform:20200000.gpio--spi:spi0.0 (devlink)
UDEV  [8470.942011] remove   /devices/virtual/devlink/platform:20200000.gpio--spi:spi0.0 (devlink)
KERNEL[8470.943013] bind     /devices/platform/soc/20204000.spi/spi_master/spi0/spi0.0 (spi)
KERNEL[8470.944010] add      /devices/platform/soc/20204000.spi/spi_master/spi0/spi0.1 (spi)
KERNEL[8470.945949] remove   /devices/virtual/devlink/platform:20101000.cprman--platform:20204000.spi (devlink)
KERNEL[8470.946695] remove   /devices/virtual/devlink/platform:20007000.dma--platform:20204000.spi (devlink)
KERNEL[8470.947408] remove   /devices/virtual/devlink/platform:20200000.gpio--platform:20204000.spi (devlink)
UDEV  [8470.949933] remove   /devices/virtual/devlink/platform:20101000.cprman--platform:20204000.spi (devlink)
KERNEL[8470.951152] bind     /devices/platform/soc/20204000.spi (platform)
KERNEL[8470.951788] add      /bus/platform/drivers/spi-bcm2835 (drivers)
KERNEL[8470.952268] add      /module/spi_bcm2835 (module)
UDEV  [8470.953742] add      /devices/platform/soc/20204000.spi (platform)
UDEV  [8470.956071] remove   /devices/virtual/devlink/platform:20007000.dma--platform:20204000.spi (devlink)
UDEV  [8470.967658] add      /devices/platform/soc/20204000.spi/spi_master/spi0 (spi_master)
UDEV  [8470.982176] remove   /devices/virtual/devlink/platform:20200000.gpio--platform:20204000.spi (devlink)
UDEV  [8470.989488] add      /bus/platform/drivers/spi-bcm2835 (drivers)
UDEV  [8471.004173] add      /module/spi_bcm2835 (module)
KERNEL[8471.226594] add      /class/spidev (class)
UDEV  [8471.238880] add      /class/spidev (class)
KERNEL[8471.240174] add      /devices/platform/soc/20204000.spi/spi_master/spi0/spi0.1/spidev/spidev0.1 (spidev)
KERNEL[8471.241757] bind     /devices/platform/soc/20204000.spi/spi_master/spi0/spi0.1 (spi)
KERNEL[8471.243266] add      /bus/spi/drivers/spidev (drivers)
UDEV  [8471.249167] add      /bus/spi/drivers/spidev (drivers)
KERNEL[8471.249934] add      /module/spidev (module)
UDEV  [8471.258036] add      /devices/platform/soc/20204000.spi/spi_master/spi0/spi0.1 (spi)
UDEV  [8471.282777] add      /module/spidev (module)
UDEV  [8471.331700] add      /devices/platform/soc/20204000.spi/spi_master/spi0/spi0.0 (spi)
UDEV  [8471.397467] add      /devices/platform/soc/20204000.spi/spi_master/spi0/spi0.0/usb1 (usb)
UDEV  [8471.413189] add      /devices/platform/soc/20204000.spi/spi_master/spi0/spi0.0/usb1/1-0:1.0 (usb)
UDEV  [8471.426507] bind     /devices/platform/soc/20204000.spi/spi_master/spi0/spi0.0/usb1/1-0:1.0 (usb)
UDEV  [8471.452175] bind     /devices/platform/soc/20204000.spi/spi_master/spi0/spi0.0/usb1 (usb)
UDEV  [8471.624578] bind     /devices/platform/soc/20204000.spi/spi_master/spi0/spi0.0 (spi)
UDEV  [8471.631928] bind     /devices/platform/soc/20204000.spi (platform)
UDEV  [8471.641903] add      /devices/platform/soc/20204000.spi/spi_master/spi0/spi0.1/spidev/spidev0.1 (spidev)
UDEV  [8471.749334] bind     /devices/platform/soc/20204000.spi/spi_master/spi0/spi0.1 (spi)

# ANd here is the output of dmesg WITH SOME NEAT STUFF ( I THINK ;P )
$ dmesg
[ 3535.914249] max3421_hcd: loading out-of-tree module taints kernel.
[ 8470.846252] OF: overlay: WARNING: memory leak will occur if overlay removed, property: /soc/spi@7e204000/status
[ 8470.846304] OF: overlay: WARNING: memory leak will occur if overlay removed, property: /soc/spi@7e204000/spidev@0/status
[ 8470.991624] max3421-hcd spi0.0: property 'maxim,vbus-en-pin' value is <1 1>
[ 8470.992549] max3421-hcd spi0.0: MAX3421 USB Host-Controller Driver
[ 8470.992594] max3421-hcd spi0.0: new USB bus registered, assigned bus number 1
[ 8470.993046] usb usb1: New USB device found, idVendor=1d6b, idProduct=0002, bcdDevice= 5.10
[ 8470.993070] usb usb1: New USB device strings: Mfr=3, Product=2, SerialNumber=1
[ 8470.993085] usb usb1: Product: MAX3421 USB Host-Controller Driver
[ 8470.993099] usb usb1: Manufacturer: Linux 5.10.63+ max3421
[ 8470.993111] usb usb1: SerialNumber: spi0.0
[ 8471.006530] max3421-hcd spi0.0: bad rev 0x00
[ 8471.008603] hub 1-0:1.0: USB hub found
[ 8471.008733] hub 1-0:1.0: 1 port detected
[ 8481.204003] max3421-hcd spi0.0: bad rev 0x00

# Quite logically, no msg(s) for 'sudo vcdbg log msg', since we're loading our module dynamically
# and R: Extra debugging can be enabled by adding dtdebug=1 to config.txt.

# Nb: digging that 'bad rev 0xnn' at: https://github.com/raspberrypi/linux/blob/rpi-5.10.y/drivers/usb/host/max3421-hcd.c
# L1375, we find the following, and it seems to me its quite logical it throws such error if no hardware is actually currently connected ..
# ( looking at the code, it tries to 'spi_rd8' ( read 8 bits from spi ? ) & then checks those against the max3421 rev number (0x12 or 0x13 )
# and if such revision number isn't found, keep looping & printing 'bad rev 0xnn' with the revision read, aka 0x00 cuz nothing's there .. )
# .. and "very hopefully" will be solved once I connect the 'modded USB Host Shield Mini', provided my mods, the pin mapping & the pins in the overlay are ok ..
max3421_spi_thread(void *dev_id)
{
	struct usb_hcd *hcd = dev_id;
	struct spi_device *spi = to_spi_device(hcd->self.controller);
	struct max3421_hcd *max3421_hcd = hcd_to_max3421(hcd);
	int i, i_worked = 1;

	/* set full-duplex SPI mode, low-active interrupt pin: */
	spi_wr8(hcd, MAX3421_REG_PINCTL,
		(BIT(MAX3421_PINCTL_FDUPSPI_BIT) |	/* full-duplex */
		 BIT(MAX3421_PINCTL_INTLEVEL_BIT)));	/* low-active irq */

	while (!kthread_should_stop()) {
		max3421_hcd->rev = spi_rd8(hcd, MAX3421_REG_REVISION);
		if (max3421_hcd->rev == 0x12 || max3421_hcd->rev == 0x13)
			break;
		dev_err(&spi->dev, "bad rev 0x%02x", max3421_hcd->rev);


# But when we check a human-readable representation of the current state of DT like this:
$ dtc -I fs /proc/device-tree
# (..)
	spi@7e204000 {
			compatible = "brcm,bcm2835-spi";
			clocks = <0x07 0x14>;
			status = "okay";
			#address-cells = <0x01>;
			interrupts = <0x02 0x16>;
			cs-gpios = <0x0a 0x08 0x01 0x0a 0x07 0x01>;
			#size-cells = <0x00>;
			dma-names = "tx\0rx";
			phandle = <0x22>;
			reg = <0x7e204000 0x200>;
			pinctrl-0 = <0x0e 0x0f>;
			dmas = <0x0b 0x06 0x0b 0x07>;
			pinctrl-names = "default";

			max3421@0 {
				maxim,vbus-en-pin = <0x01 0x01>;
				compatible = "maxim,max3421";
				interrupt-parent = <0x0a>;
				interrupts = <0x19 0x02>;
				phandle = <0x83>;
				reg = <0x00>;
				pinctrl-0 = <0x82>;
				spi-max-frequency = <0x1312d00>;
				pinctrl-names = "default";
			};

			spidev@1 {
				compatible = "spidev";
				#address-cells = <0x01>;
				#size-cells = <0x00>;
				phandle = <0x64>;
				reg = <0x01>;
				spi-max-frequency = <0x7735940>;
			};

			spidev@0 {
				compatible = "spidev";
				status = "disabled";
				#address-cells = <0x01>;
				#size-cells = <0x00>;
				phandle = <0x63>;
				reg = <0x00>;
				spi-max-frequency = <0x7735940>;
			};
		};
(..)

# So, since it smells good, let's unload the dynamically loaded overlay
$ sudo dtoverlay -r max3421-hcd
# --> no errors ( nor logs .. )
# AS the RPi foundation doc states, modules won't unload by themselves for dynamically loaded DTs
$ lsmod  | grep max34
#max3421_hcd            24576  0
# As stated above, the 'rmmod -a' cmd errored on the 1st try for me ( wrong -param ? no '-a' is supported ..  ), yet the following once removed it from lsmod 
$ sudo rmmod max3421_hcd'
# no errors
$ lsmod  | grep max34
# nothing to see here

# So, logical net step(s):
# copying the overlay's .dtbo to /boot/overlays/
sudo cp ./max3421-hcd.dtbo /boot/overlays/
# Making sure it's now there ( but why shouldn't it ? ):
$ ls /boot/overlays/ | grep max342
#max3421-hcd.dtbo

# Oh, I forgot to run the following cmd to get the state of the pins once the DT overlay or the driver has loaded ( easier done when loading the module dynamically )
$ raspi-gpio get

# So, now editing /boot/config.txt & adding the needded 'dtoverlay' "firmware directive" ( correct name ? )
# R: the current overlay has his 'overrides' commented out, so no params should be able to be passed from the overlay to the driver
# Also, to try with dynamic overlay loading, we should pass as a one-liner what could be split in 'dtoverlay=' & 'dtparam=' in /boot/config.txt
# R: previous findings related: https://github.com/stephaneAG/RPi/blob/master/max3421_spiUsbHost.md
# mandatory ---> Nb: no yet sure of how to use this dtparam after other 'dtoverlay=' have been set, currently only de-commenting the one already present at near top of /boot/config.txt
#$ echo "dtparam=spi=on" | sudo tee -a /boot/config.txt
# trying without 1st ..
#$ dtoverlay=spi0-1cs
#( or dtoverlay -h spi0-2cs )
$ echo "dtoverlay=max3421-hcd" | sudo tee -a /boot/config.txt

# trying to 'force' chip select for below spi peripheral ?
# Set GPIO8 to be an output set to 0 ( OutPut, DriveLow ), see: https://www.raspberrypi.com/documentation/computers/config_txt.html#gpio
# trying without 1st & tweaking if needed with raspi-gpio ;p
#gpio=8=op,dl

# Now, halting temporarly the RPi Zero W, jsut long enough for me to connect the 'modded USB Host Shield Mini' to it over the pins showed in the 'pin mapping' visual ( github repo )
$ sudo halt -p
# --> once done, the green LED is no longer lit ;p

# Booting ...
# connecting over ssh .. ok
# testing if our g_serial gadget still works .. ok

# -- Checking lsmod .. ok
$ lsmod | grep max342
# max3421_hcd            24576  0 

# -- Checking dmesg .. ok
$ dmesg

======== START OF DMESG ========
[    0.000000] OF: fdt: Machine model: Raspberry Pi Zero W Rev 1.1
( .. )
[    0.000000] Kernel command line: coherent_pool=1M 8250.nr_uarts=0 snd_bcm2835.enable_compat_alsa=0 snd_bcm2835.enable_hdmi=1 video=Composite-1:720x480@60i smsc95xx.macaddr=B8:27:EB:49:B5:F9 vc_mem.mem_base=0x1ec00000 vc_mem.mem_size=0x20000000  console=ttyS0,115200 console=tty1 root=PARTUUID=f6b8d2a5-02 rootfstype=ext4 fsck.repair=yes rootwait modules-load=dwc2,g_serial
( .. )
[    0.116410] pinctrl core: initialized pinctrl subsystem
( .. )
[    0.257349] usbcore: registered new interface driver usbfs
[    0.257516] usbcore: registered new interface driver hub
[    0.257674] usbcore: registered new device driver usb
[    0.260686] clocksource: Switched to clocksource timer
( .. )
[    2.546086] usbcore: registered new interface driver smsc95xx
[    2.549744] dwc_otg: version 3.00a 10-AUG-2012 (platform bus)
[    2.553854] dwc_otg: FIQ enabled
[    2.553877] dwc_otg: NAK holdoff enabled
[    2.553891] dwc_otg: FIQ split-transaction FSM enabled
[    2.553918] Module dwc_common_port init
[    2.554430] usbcore: registered new interface driver usb-storage
[    2.558597] mousedev: PS/2 mouse device common for all mice
[    2.564287] bcm2835-wdt bcm2835-wdt: Broadcom BCM2835 watchdog timer
[    2.571413] sdhci: Secure Digital Host Controller Interface driver
[    2.575081] sdhci: Copyright(c) Pierre Ossman
[    2.579573] mmc-bcm2835 20300000.mmcnr: could not get clk, deferring probe
[    2.584466] sdhost-bcm2835 20202000.mmc: could not get clk, deferring probe
[    2.588705] sdhci-pltfm: SDHCI platform and OF driver helper
[    2.593545] ledtrig-cpu: registered to indicate activity on CPUs
[    2.597905] hid: raw HID events driver (C) Jiri Kosina
[    2.601935] usbcore: registered new interface driver usbhid
[    2.605595] usbhid: USB HID core driver
( .. )
[    2.711745] sdhost: log_buf @ (ptrval) (8bd42000)
[    2.752102] mmc1: queuing unknown CIS tuple 0x80 (2 bytes)
[    2.757527] mmc1: queuing unknown CIS tuple 0x80 (3 bytes)
[    2.762793] mmc1: queuing unknown CIS tuple 0x80 (3 bytes)
[    2.766226] mmc0: sdhost-bcm2835 loaded - DMA enabled (>1)
[    2.773776] of_cfs_init
[    2.777294] of_cfs_init: OK
[    2.803542] Waiting for root device PARTUUID=f6b8d2a5-02...
[    2.807628] mmc1: queuing unknown CIS tuple 0x80 (7 bytes)
[    2.846755] mmc0: host does not support reading read-only switch, assuming write-enable
[    2.852031] mmc0: Problem switching card into high-speed mode!
[    2.856549] mmc0: new SDHC card at address 0001
[    2.861794] mmcblk0: mmc0:0001 SD16G 29.1 GiB
[    2.869447]  mmcblk0: p1 p2
[    2.914332] EXT4-fs (mmcblk0p2): mounted filesystem with ordered data mode. Opts: (null)
[    2.921136] VFS: Mounted root (ext4 filesystem) readonly on device 179:2.
[    2.940925] devtmpfs: mounted
( .. )
[    2.971211] mmc1: new high speed SDIO card at address 0001
( .. )
[    9.993886] systemd[1]: Mounting FUSE Control File System...
[   10.210051] dwc2 20980000.usb: supply vusb_d not found, using dummy regulator
[   10.214756] systemd[1]: Mounting Kernel Configuration File System...
[   10.231351] dwc2 20980000.usb: supply vusb_a not found, using dummy regulator
[   10.333138] dwc2 20980000.usb: EPs: 8, dedicated fifos, 4080 entries in SPRAM
[   10.377484] systemd[1]: Started File System Check Daemon to report status.
[   10.550996] systemd[1]: Finished File System Check on Root Device.
[   10.595733] g_serial gadget: Gadget Serial v2.4
[   10.599796] g_serial gadget: g_serial ready
[   10.635769] systemd[1]: Mounted FUSE Control File System.
[   10.641263] dwc2 20980000.usb: bound driver g_serial
[   10.763372] systemd[1]: Finished Load Kernel Modules.
[   10.813991] systemd[1]: Mounted Kernel Configuration File System.
[   10.899013] dwc2 20980000.usb: new device is high-speed
[   10.922784] systemd[1]: Starting Remount Root and Kernel File Systems...
[   10.954131] dwc2 20980000.usb: new device is high-speed
[   10.975961] dwc2 20980000.usb: new address 33
[   11.066992] systemd[1]: Starting Apply Kernel Variables...
[   11.434730] systemd[1]: Started Journal Service.
[   12.211217] EXT4-fs (mmcblk0p2): re-mounted. Opts: (null)
( .. )
[   21.722454] usbcore: registered new interface driver brcmfmac
( .. )
[   32.243739] max3421_hcd: loading out-of-tree module taints kernel.
[   32.245322] max3421-hcd spi0.0: property 'maxim,vbus-en-pin' value is <1 1>
[   32.256152] max3421-hcd spi0.0: MAX3421 USB Host-Controller Driver
[   32.256348] max3421-hcd spi0.0: rev 0x13, SPI clk 20000000Hz, bpw 8, irq 160
[   32.256504] max3421-hcd spi0.0: new USB bus registered, assigned bus number 1
[   32.260151] usb usb1: New USB device found, idVendor=1d6b, idProduct=0002, bcdDevice= 5.10
[   32.260174] usb usb1: New USB device strings: Mfr=3, Product=2, SerialNumber=1
[   32.260190] usb usb1: Product: MAX3421 USB Host-Controller Driver
[   32.260203] usb usb1: Manufacturer: Linux 5.10.63+ max3421
[   32.260215] usb usb1: SerialNumber: spi0.0
[   32.329376] hub 1-0:1.0: USB hub found
[   32.340799] hub 1-0:1.0: 1 port detected
[   32.610931] usb 1-1: new full-speed USB device number 2 using max3421-hcd
[   32.818530] usb 1-1: New USB device found, idVendor=8644, idProduct=8003, bcdDevice= 1.00
[   32.818562] usb 1-1: New USB device strings: Mfr=1, Product=2, SerialNumber=3
[   32.818578] usb 1-1: Product: USB Flash Disk                
[   32.818591] usb 1-1: Manufacturer: General                       
[   32.818604] usb 1-1: SerialNumber: 0333000000018C5B
[   32.820467] usb-storage 1-1:1.0: USB Mass Storage device detected
[   32.838465] scsi host0: usb-storage 1-1:1.0
( .. )
[   33.853787] scsi 0:0:0:0: Direct-Access     General  USB Flash Disk   1.00 PQ: 0 ANSI: 2
[   33.869255] sd 0:0:0:0: [sda] 3913728 512-byte logical blocks: (2.00 GB/1.87 GiB)
[   33.878192] sd 0:0:0:0: [sda] Write Protect is off
[   33.878226] sd 0:0:0:0: [sda] Mode Sense: 03 00 00 00
[   33.881927] sd 0:0:0:0: [sda] No Caching mode page found
[   33.881958] sd 0:0:0:0: [sda] Assuming drive cache: write through
[   34.018862]  sda: sda1
[   34.042444] sd 0:0:0:0: [sda] Attached SCSI removable disk
[   35.452483] brcmfmac: brcmf_cfg80211_set_power_mgmt: power save enabled
( .. )
[   37.179931] usbcore: registered new interface driver uas
[   37.735342] sd 0:0:0:0: Attached scsi generic sg0 type 0
( .. )
======== END OF DMESG ========

# -- Checking overlay .. ok
#human-readable representation of the current state of DT like this:
$ dtc -I fs /proc/device-tree
======== START OF HUMAN READABLE DT ========
spi@7e204000 {
			compatible = "brcm,bcm2835-spi";
			clocks = <0x07 0x14>;
			status = "okay";
			#address-cells = <0x01>;
			interrupts = <0x02 0x16>;
			cs-gpios = <0x0a 0x08 0x01 0x0a 0x07 0x01>;
			#size-cells = <0x00>;
			dma-names = "tx\0rx";
			phandle = <0x22>;
			reg = <0x7e204000 0x200>;
			pinctrl-0 = <0x0e 0x0f>;
			dmas = <0x0b 0x06 0x0b 0x07>;
			pinctrl-names = "default";

			max3421@0 {
				maxim,vbus-en-pin = <0x01 0x01>;
				compatible = "maxim,max3421";
				interrupt-parent = <0x0a>;
				interrupts = <0x19 0x02>;
				phandle = <0x84>;
				reg = <0x00>;
				pinctrl-0 = <0x83>;
				spi-max-frequency = <0x1312d00>;
				pinctrl-names = "default";
			};

			spidev@1 {
				compatible = "spidev";
				#address-cells = <0x01>;
				#size-cells = <0x00>;
				phandle = <0x64>;
				reg = <0x01>;
				spi-max-frequency = <0x7735940>;
			};

			spidev@0 {
				compatible = "spidev";
				status = "disabled";
				#address-cells = <0x01>;
				#size-cells = <0x00>;
				phandle = <0x63>;
				reg = <0x00>;
				spi-max-frequency = <0x7735940>;
			};
		};
======== END OF HULAN READABLE DT ========


# -- Checking lsusb .. ok 
$ lsusb
#Bus 001 Device 002: ID 8644:8003 Intenso GmbG Micro Line --> HAHA, THE USB KEY THAT'S PLUGGED IN ;P
#Bus 001 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub

# Checking if the USB KEY plugged in actually appears using 'fdisk':
$ sudo fdisk -l
# ( .. ) --> /dev/ramw<n> & /dev/mmcblk0 ( & its /dev/mmcblk0p1 & p2 )
Disk /dev/sda: 1.87 GiB, 2003828736 bytes, 3913728 sectors
Disk model: USB Flash Disk  
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0x00000000

Device     Boot Start     End Sectors  Size Id Type
/dev/sda1          40 3913727 3913688  1.9G  b W95 FAT32


# 'Quick' simplest try ( before I move onto replacing g_serial with libcomposite & making my USB Gadget's mass_storage using the external USB KEY as storage ( if available ) [ or fallback to little portion on a otherwise read--only SD ? ]
# create a dir to mount the device to ( from https://github.com/stephaneAG/RPi/blob/master/UsbGadget.md )
$ sudo mkdir /mnt/usb_share
# R: we DON'T go the same way as in https://github.com/stephaneAG/RPi/blob/master/UsbGadget.md:
# instead, we follow part of the way exposed at https://github.com/stephaneAG/RPi/blob/master/UsbGadget_LibComposite.md
# my best guess:
$ mount -t vfat /dev/sda1 /mnt/usb_share
# OR Remember syntax for libcomposite & DD created backing store img 'subdisk.img'
#FILE=/home/pi/usbdisk.img
#mkdir -p ${FILE/img/d} ----> this 'feels weird', is the syntax correct ? ( I know it worked on my mac os laptop, but didn't well on a windows one ( altghough it could come from the way I did partition part of the SD as the backing storage .. )
#mount -o loop,ro, -t vfat $FILE ${FILE/img/d} # FOR IMAGE CREATED WITH DD
# should work ok for libcomposite version ?
#$ mount -o loop,ro, -t vfat /dev/sda1 /mnt/usb_share

# So, currently trying the following:
$ sudo mount -t vfat /dev/sda1 /mnt/usb_share
# no errors ..
$ ls /mnt/usb_share/
 658lines_later__in_this_file_only___yup.png   AbletonLaurine  'Devis&Facture_n1'  'Devis&Facture_n3'   StephaneAG_Passport.png  'Talks! - AirLiquide_iLab_ Moldr.pdf'
 ADS1000                                       DAV             'Devis&Facture_n2'  'Devis&Facture_n4'   Talk_images               nicolas_ca_ressemble_a_ca_pour_obtenir_le_PDF.png
# YESSSSSSS !!!
$ sudo touch /mnt/usb_share/testOf05012022_throughMax3421spiUsbHostShieldMiniModdedAndDtoverlayAndDriver
# the above added the said file, so we have actual write rights ( using sudo ) to it ( R: why ? cuz I got 'mount: /mnt/usb_share: must be superuser to use mount.' earlier .. )

# Once we've done all we wanted, let's unmount it
$ sudo umount /dev/sda1
# To verify, 'usb_share' is now empty
$ ls /mnt/usb_share/

# .. and let's shut it down for today .. ;p
$ sudo halt -p

# Digging if I can find where to control the GPOUTx located on the MAX3421 ( it if's made available by the driver I guess ? )
$ ls /sys/class/spi_master/spi0/spi0.0/usb1/
# I see stuff there, but no clue on what's what ..
# 

# Launching another ssh session in another terminal tab to monitor events from now on ..
$ cd /home/pi/max3421-hcd/udevLogs/
$ udevadm monitor > ./udevlogs.txt
```

#### 5: Well, if you followed that far, you should now have a working USB Host port over the RPi's GPIO PI0C0 :)
