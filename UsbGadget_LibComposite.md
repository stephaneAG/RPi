R: mainly from http://www.isticktoit.net/?p=1383

## Step 1 - kernel stuff
We're supposed to use at least the 4.4 kernel ( as of feb 2016 not installed in the def Raspbian img ),
yet I'm writing these lines on 16/12/2021, so it should be ok for latests Raspbians

- enable the 'special' device tree overlay by adding it to '/boot/config.txt' & '/etc/modules':
```
echo "dtoverlay=dwc2" | sudo tee -a /boot/config.txt
echo "dwc2" | sudo tee -a /etc/modules
```
- enable the 'licomposite' driver:
```
echo "libcomposite" | sudo tee -a /etc/modules
```

## Step 2 - configure the gadget(s)
Ethernet, keyboard/mouse/joystick, mass storage, serial, ..
All the config is done using 'ConfigFS', a virtual fs in /sys that's auto-mounted on startup
R: the original tut is based on an example called 'USBArmory' ( start point: https://github.com/ckuethe/usbarmory/wiki/USB-Gadgets )

- creating the config script:
The config is 'volatile', so must be run on each startup
For this reason, we create a 'isticktoit_usb' file in /usr/bin & make it executable
```
sudo touch /usr/bin/isticktoit_usb #create the file
sudo chmod +x /usr/bin/isticktoit_usb #make it executable
sudo nano /usr/bin/isticktoit_usb #edit the file
```
After doing so, we have to run this script on startup
( for best perfs, using a 'systemd unit file', but for quick testing, 'rc.local' ( part of the 'old' sysvinit system )
should be fine since still executed on the RPi by default.
We add the following RIGHT BEFORE the ending 'exit 0'
```
/usr/bin/isticktoit_usb # libcomposite configuration
```

- creating the Gadget:
This is a GLOBAL configuration, so no matter how many USB features the RPi uses :)
( Feel free to change serial number, manufacturer & product name )
```
#!/bin/bash
cd /sys/kernel/config/usb_gadget/
mkdir -p isticktoit
cd isticktoit
echo 0x1d6b > idVendor # Linux Foundation
echo 0x0104 > idProduct # Multifunction Composite Gadget
echo 0x0100 > bcdDevice # v1.0.0
echo 0x0200 > bcdUSB # USB2
mkdir -p strings/0x409
echo "fedcba9876543210" > strings/0x409/serialnumber
echo "Tobias Girstmair" > strings/0x409/manufacturer
echo "iSticktoit.net USB Device" > strings/0x409/product
mkdir -p configs/c.1/strings/0x409
echo "Config 1: ECM network" > configs/c.1/strings/0x409/configuration
echo 250 > configs/c.1/MaxPower
# Add functions here 
# ( .. )
# see gadget configurations below
# End functions
ls /sys/class/udc > UDC
```

Now, all that's left on this config is adding the desired interfaces

## Interface: Serial Adapter
Useful for debugging & easiest to setup
```
# Add functions here
mkdir -p functions/acm.usb0
ln -s functions/acm.usb0 configs/c.1/
# End functions
```
Then, enable a console on the USB serial: ```sudo systemctl enable getty@ttyGS0.service```
Finally, to connect to the RPi: ```sudo screen /dev/ttyACM0 115200```
( if screens terminates auto, maybe it's needed to change the device file, check: ```dmesg|grep "USB ACM device"```

## Interface: Ethernet Adapter
```
# Add functions here
mkdir -p functions/ecm.usb0
# first byte of address must be even
HOST="48:6f:73:74:50:43" # "HostPC"
SELF="42:61:64:55:53:42" # "BadUSB"
echo $HOST > functions/ecm.usb0/host_addr
echo $SELF > functions/ecm.usb0/dev_addr
ln -s functions/ecm.usb0 configs/c.1/
# End functions
ls /sys/class/udc > UDC
#put this at the very end of the file:
ifconfig usb0 10.0.0.1 netmask 255.255.255.252 up
route add -net default gw 10.0.0.2
```
If problems with the automatic connection, try: ```dmesg|grep cdc_ether```
Once done using the previous line, plug the interface name into the following line: ```sudo ifconfig enp0s20u1i2 10.0.0.2 netmask```
Finally, connect via ssh to the RPi: ```ssh 10.0.0.1 -l pi```

## Interface: Keyboard / Mouse / Joystick ( HID )
```
# Add functions here
mkdir -p functions/hid.usb0
echo 1 > functions/hid.usb0/protocol
echo 1 > functions/hid.usb0/subclass
echo 8 > functions/hid.usb0/report_length
echo -ne \\x05\\x01\\x09\\x06\\xa1\\x01\\x05\\x07\\x19\\xe0\\x29\\xe7\\x15\\x00\\x25\\x01\\x75\\x01\\x95\\x08\\x81\\x02\\x95\\x01\\x75\\x08\\x81\\x03\\x95\\x05\\x75\\x01\\x05\\x08\\x19\\x01\\x29\\x05\\x91\\x02\\x95\\x01\\x75\\x03\\x91\\x03\\x95\\x06\\x75\\x08\\x15\\x00\\x25\\x65\\x05\\x07\\x19\\x00\\x29\\x65\\x81\\x00\\xc0 > functions/hid.usb0/report_desc
ln -s functions/hid.usb0 configs/c.1/
# End functions
```
Then, the simplest way to send keystrokes is by echoing the HID packets to the device file:
```
sudo su
echo -ne "\0\0\x4\0\0\0\0\0" > /dev/hidg0 #press the A-button
echo -ne "\0\0\0\0\0\0\0\0" > /dev/hidg0 #release all keys
```
Nb: the original author of the tut offers some code for more practical handling of the above keystroke sending at https://git.gir.st/sendHID.git
```
make #compile the program
echo -n "hello world!" | sudo ./scan /dev/hidg0 1 2 # write piped sring over HID
# R: param 1: '1' == US-layout || '2' == German/Australian layout, param 2: chars not available on kbd ( .. )
```

## Interface: Mass Storage
"Somewhat difficult":
We can't share the RPi's partition with the host computer, only a disk mimage file
We can create a "small" one to store stuff on it
Tef Nb 1: test with an external drive ..
Tef Nb 2: test how this 'mixes' with using 'fstab' ( & possibly samba )
- 1st, we make a disk ( see "USBGadget.md" on this repo ) using the followinf commands here:
```
dd if=/dev/zero of=~/usbdisk.img bs=1024 count=1024
mkdosfs ~/usbdisk.img
```
- then, we edit our Gadget configuration:
```
# Add functions here
FILE=/home/pi/usbdisk.img
mkdir -p ${FILE/img/d}
# mount -o loop,ro,offset=1048576 -t ext4 $FILE ${FILE/img/d} # FOR OLD WAY OF MAKING THE IMAGE
mount -o loop,ro, -t vfat $FILE ${FILE/img/d} # FOR IMAGE CREATED WITH DD
mkdir -p functions/mass_storage.usb0
echo 1 > functions/mass_storage.usb0/stall
echo 0 > functions/mass_storage.usb0/lun.0/cdrom
echo 0 > functions/mass_storage.usb0/lun.0/ro
echo 0 > functions/mass_storage.usb0/lun.0/nofua
echo $FILE > functions/mass_storage.usb0/lun.0/file
ln -s functions/mass_storage.usb0 configs/c.1/
# End functions
```
A FAT32 formatted removable drive should show up
To accessthe file stored from the RPi, we can unmount it completely ( in host 1st, then RPi ), then remount it somewhere else
