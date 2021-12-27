R: last updated: 14/12/2021
R: digging infos from the following sources:
- https://github.com/raspberrypi/firmware/blob/master/boot/overlays/README
- http://www.linux-usb.org/gadget/
- USB OR SD: http://www.linux-usb.org/gadget/file_storage.html
- https://web.archive.org/web/20211123174909/http://linux-sunxi.org/USB_Gadget
- https://www.google.com/search?q=shared+backing+storage+g_mass_storage&rlz=1C5CHFA_enFR920FR920&oq=shared+backing+storage+g_mass_storage&aqs=chrome..69i57.11645j1j7&sourceid=chrome&ie=UTF-8
- https://learn.adafruit.com/turning-your-raspberry-pi-zero-into-a-usb-gadget/other-modules
- https://magpi.raspberrypi.com/articles/pi-zero-w-smart-usb-flash-drive
- https://forums.raspberrypi.com/viewtopic.php?t=216810
- https://randomnerdtutorials.com/raspberry-pi-zero-usb-keyboard-hid/
- https://github.com/ckuethe/usbarmory/wiki/USB-Gadgets
- NEAT XPLANATION: http://www.isticktoit.net/?p=1383
- VERY NICE: https://wiki.tizen.org/USB/Linux_USB_Layers/Configfs_Composite_Gadget
- ALSO: https://wiki.tizen.org/USB/Linux_USB_Layers/Configfs_Composite_Gadget/General_configuration
- RECAP: https://www.kernel.org/doc/Documentation/usb/gadget_configfs.txt
- https://github.com/raspberrypisig/pizero-usb-hid-keyboard
- https://git.gir.st/hardpass.git/blob/HEAD:/init_usb.sh
- https://usb.org/sites/default/files/hut1_2.pdf
- https://stackoverflow.com/questions/21606991/custom-hid-device-hid-report-descriptor
- https://www.framboise314.fr/un-point-sur-le-device-tree/#Overlays_et_configtxt
- https://github.com/raspberrypi/linux/issues/1212#issuecomment-165148405
- https://www.kernel.org/doc/Documentation/usb/gadget-testing.txt
- https://github.com/alexellis/docker-arm/blob/master/OTG.md
- https://github.com/Raspberryy/Emulated_USB_Printer

Sources closely related & also how to implm on RPi4:
- https://howchoo.com/pi/raspberry-pi-gadget-mode
- USB Gadget RNDIS/ECM + 2xACM: https://pastebin.com/VtAusEmf
- https://github.com/lucaong/nerves_rpi4_hid_gadget_poc
- https://gist.github.com/ianfinch/08288379b3575f360b64dee62a9f453f
- https://github.com/kmpm/rpi-usb-gadget
- https://news.ycombinator.com/item?id=21430039
- https://stackoverflow.com/questions/66184576/usb-hid-data-initialisation
- https://www.hardill.me.uk/wordpress/2019/11/02/pi4-usb-c-gadget/
- https://www.hardill.me.uk/wordpress/2017/01/23/raspberry-pi-zero-gadgets/
- https://forum.armbian.com/topic/1759-usb-gadget-built-for-h3-processor-on-top-of-gadgetfs/
- https://learn.adafruit.com/resizing-raspberry-pi-boot-partition/bonus-shrinking-images
- https://devotics.fr/construire-camera-ip-raspbery-pi-zero-w/

Main goal(s):

- mass storage ( & external usb key for RPi4, + gpio usb host adapter board for RPi0 )
- HID
- ?
- camera

## DEBUG
from 'https://gist.github.com/gbaman/50b6cca61dd1c3f88f41'
we can read that ```Need to now pick which module you want to use from the list above, for example for ethernet echo "g_ether" | sudo tee -a /etc/modules. You can only pick one of the above modules to use at a time.```
Hence if cmdline.txt already contains 'modules-load=dwc2,g_ether', this may prevent ```sudo modprobe g_mass_storage file=/piusb.bin stall=0 ro=1``` to work ?

Also, at 'https://forums.raspberrypi.com/viewtopic.php?t=265754' we can read that ```There are kernel modules to emulate a number of USB devices and libcomposite for situations where a combination of devices is needed but not provided by an existing module.```
Brief guide provided:
1: /boot/config.txt ```dtoverlay=dwc2,dr_mode=peripheral```
Ommit ",dr_mode=peripheral" if you're on a zero(w) and want to be able to hot swap between roles.
R: MY config.txt looks like:
```
dtoverlay=vc4-kms-v3d # 'Enable DRM VC4 V3D driver' ( present for newer RPi OS installations it seems )
..
dtoverlay=dwc2 # current ethernet gadget implm -> can be 'hot-swappable' for roles :)
..
dtoverlay=hifiberry-dac # Pimoroni PirateAudio
```
2: /boot/cmdline.txt ```modules-load=dwc2,<gadget module of choice>```
R: MY cmdline.txt looks like
``` .. modules-load=dwc2,g_ether```
So, maybe ```sudo modprobe -r g_ether``` & then retry ```sudo modprobe g_mass_storage file=/piusb.bin stall=0 ro=1```
Nb: after testing, 'sudo modprobe -r g_ether' removes the 'RNDIS/EthernetGadget' present in Mac OS lapotop >Network settings ..
Nb2: & after issuing 'sudo modprobe g_mass_storage file=/piusb.bin stall=0 ro=1' over ssh, a 'NONAME' device appeared ;P
Nb3: if desiring a read/write device, use the following instead: ```sudo modprobe g_mass_storage file=/piusb.bin stall=0 removable=y ro=0```
3: if using 'libcomposite', do the configuration after boot (systemd service, rc.local, crontab, etc).


## Mass Storage ( original 'source' tutorial: https://magpi.raspberrypi.com/articles/pi-zero-w-smart-usb-flash-drive )

- 1: make sure to have 'dtoverlay=dwc2' in '/boot/config.txt'
- 2: ```sudo nano /etc/modules``` & append 'dwc2' after 'i2c-dev' line ( nb: I didn't have the 'i2c-dev' line
- 3: create a 'container file' ( Nb: a more advanced route using so-called 'backing storage' & partition(s): http://www.linux-usb.org/gadget/file_storage.html )
The 'container' file will act as the storag medium on our SD card ( for the 'simple' route )
Check the available space on the SD card beforehand using ```df -h```
Then ( ex for 1Gb ) ```sudo dd bs=1M if=/dev/zero of=/piusb.bin count=1024```
- 4: format as FAT32 filesystem so that most devices ( ex: TV ) can understand it: ```sudo mkdosfs /piusb.bin -F 32 -I```
- 4bis: specify a "volume label" while creating the fs: ```sudo mkdosfs -n "RASPBERRYPI" /vfat.fs.bin ... ```
- 5: mount the 'container' file so we can DL some test files:
We create a directory on which we can mount the fs ```sudo mkdir /mnt/usb_share```
Then, we add this to 'fstab', the conf file that records our available disk partitions ```sudo nano /etc/fstab```
And we append the following to the EOF: ```/piusb.bin /mnt/usb_share vfat users,umask=000 0 2```
This line added to 'fstab' allows the USB fs to be error-checked & auto-mounted at boot time
Instead of rebooting, we can manually reload 'fstab' using ```sudo mount -a```

- 6: download a test file:
```cd /mnt/usbshare````
```wget https://download.blender.org/peach/bigbuckbunny_movies/big_buck_bunny_720p_stereo.avi```
Once the DL is complete, run the following cmd to flush any cached data to the disk: ```sync````

- 7: test mass storage mode ( Nb/R: nothing appeared on my mac doing so :/ .. ):
enable the USB Mass Storage mode: ```sudo modprobe g_mass_storage file=/piusb.bin stall=0 ro=1```
once satisfied, try a dismount: ```sudo modprobe -r g_mass_storage```
( should trigger a 'USB disconnected' popup on the device it was connected to )

- 8: providing network access ( Samba ) to the /mnt/usb_share dir created earlier:
```
sudo apt-get update
sudo apt-get install samba winbind -y
```
Once done with the above lines, configure a Samba network share, without login/passwd ( see wiki.samba.org for more security ):
```sudo nano /etc/samba/smb.conf```
Append the followin at the EOF:
```
[usb]
browseable = yes
path = /mnt/usb_share
guest ok = yes
read only = no
create mask = 777
```
Then, restart the Samba service for the change to take effect ```sudo systemctl restart smbd.service```
From a mac, the device will appear in the Finder sidebar & is also accessible via server adress ( cmd+K) ```smb://raspberrypi```
( R: actually, 'Network' appears in the Finder sidebar, where we can find 'RASPBERRYPI' > 'usb' > <our tests files> )
  
- 9: ( optional ): auto USB device reconnect:
The original tuo was about forcing a TV to detect any change made over the network ( ex: file or dir creations/deletion ),
that needed to be tricked into thinking the USB device had been removed & re-inserted.
To do so, a python lib ( 'watchdog' -> magpi.cc/2SLL1fi ) designed for monitorng fs evts & repsonding to them.
To install it: ```sudo pip3 install watchdog```
The following starts a timer whenever stg changes in the shared dir & resets the USB conn after 30 secs of inacitivty atfer a change.
```
cd /usr/local/share
sudo wget http://rpf.io/usbzw -O usbshare.py
sudo chmod +x usbshare.py
```
- 10: ( optional ): background service ( that autostarts at boot time ):
```
cd /etc/systemd/system
sudo nano usbshare.service
```
Add the following to the file created above:
```
[Unit]
Description=USB Share Watchdog

[Service]
Type=simple
ExecStart=/usr/local/share/usb_share.py
Restart=always

[Install]
WantedBy=multi-user.target
```
Then, we register the servic, enable it & set it running:
```
sudo systemctl daemon-reload
sudo systemctl enable usbshare.service
sudo systemctl start usbshare.service
```
Now, whenever copying files over this network share, the USB should auto reconnect to the TV after 30 secs of inacivity
