R: the following links may get quite handy when digging the possible ways to do stuff ( while using or not USB gadget, but mostly for that ;p )



## USB Gadgets & Cie
- General 'old school': https://www.kernel.org/doc/Documentation/usb/gadget_configfs.txt
- General: https://wiki.tizen.org/USB/Linux_USB_Layers/Configfs_Composite_Gadget
- Mass Storage: https://wiki.tizen.org/USB/Linux_USB_Layers/Configfs_Composite_Gadget/Usage_eq._to_g_mass_storage.ko
- MIDI: https://unix.stackexchange.com/questions/303183/midi-linux-gadget-module-g-midi-with-ipad-on-raspberry-pi
- MIDI: https://www.youtube.com/watch?v=N4yUduOqR3M&t=242s
- MIDI: https://github.com/TaylorTMusic/MIDI2GPIO

## BEING CURRENTLY ( as of 17/12/2021 ) DIGGED FOR USB GADGET IMPLM ON RPi0/(A+/B)/4/CM
- https://git.gir.st/hardpass.git/blob/HEAD:/init_usb.sh
- https://github.com/ckuethe/usbarmory/wiki/USB-Gadgets
- DIGG HOW TO REMOVE "USD" & CIE: https://github.com/larsks/systemd-usb-gadget
- DR_MODE ( aka salve on RPIA/B !!! ?! :p ): https://www.kernel.org/doc/Documentation/devicetree/bindings/usb/generic.txt
- HOW TO SET A SPECIFIC "USB DUAL MODE ROLE": https://gist.github.com/gbaman/50b6cca61dd1c3f88f41#gistcomment-2109703
- SAME AS ABOVE REGARDING A CM4: https://forums.raspberrypi.com/viewtopic.php?t=295238
- USB Gadget LibComposite HID+MassStorage working on windaube: https://forums.raspberrypi.com/viewtopic.php?t=260107

## TO DIGG TO FIX THE ETHERNET NOT RECOGNIZED STUFF FOR LIBCOMPOSITE
- https://hackaday.io/project/19471-hackpi/details
- https://hackaday.io/project/19471/instructions
- https://github.com/wismna/HackPi/blob/master/rc.local
- https://raspberrypi.stackexchange.com/questions/64112/macos-not-discovering-raspberry-pi-zero-as-an-usb-ethernet-device
- LONGER READ WORTH STG: https://jon.sprig.gs/blog/post/2243
- 

## Ifconfig
- https://unix.stackexchange.com/questions/312920/usb0-network-interface-doesnt-get-up

## Samba stuff
- http://emery.claude.free.fr/nas-samba.html
- https://wiki.samba.org/index.php/Configure_Samba_to_Bind_to_Specific_Interfaces
- https://www.samba.org/~tpot/articles/multiple-interfaces.html

## Fstab
- https://debian-facile.org/doc:systeme:fstab

## Mount
- http://manpagesfr.free.fr/man/man8/mount.8.html
- LONG READ MAYBE WORTH IT: https://stackoverflow.com/questions/7878707/how-to-unmount-a-busy-device
- https://forums.raspberrypi.com/viewtopic.php?t=32682
- https://stackoverflow.com/questions/7878707/how-to-unmount-a-busy-device
- 

## Eject USB devices ( not 'just' unmount )
- https://raspberrypi.stackexchange.com/questions/14843/how-to-eject-usb-device-on-raspberry-pi-not-just-unmount
My humble personal tries ( on 16/12/2021, RPi0w1.1 )
```
/sys/kernel/config/usb_gadget/tefrpiusbcombo/functions/mass_storage.usb0/lun.0/file
/sys/kernel/config/usb_gadget/tefrpiusbcombo/configs/c.1/mass_storage.usb0/lun.0/file

sudo su
# possible "Device or resource busy" error msg
echo "" > /sys/kernel/config/usb_gadget/tefrpiusbcombo/functions/mass_storage.usb0/lun.0/file
echo "/pisusb.bin" > ..


----

# possible "target is busy" error msg
sudo umount /dev/loop0
# OR
sudo umount /mnt/usb_share
sudo umount -f ..

----

cat /sys/kernel/config/usb_gadget/tefrpiusbcombo/UDC
20980000.usb

# Disconnect & reconnect the whole composite device, no error msgs ;D
echo "" > /sys/kernel/config/usb_gadget/tefrpiusbcombo/UDC
echo "20980000.usb" > /sys/kernel/config/usb_gadget/tefrpiusbcombo/UDC
```

