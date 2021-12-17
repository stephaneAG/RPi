<img width="60px" src="https://www.raspberrypi.org/wp-content/themes/mind-control/images/logo-black.png"></img>

Some handy Raspberry Pi stuff ;)

## Datasheets/Schematics
https://datasheets.raspberrypi.com/

Questionning the USB 'roles' & whre/how it connects:
- schematic for USB interface question: https://forums.raspberrypi.com/viewtopic.php?t=217858
- https://raspberrypi.stackexchange.com/questions/127415/are-schematics-available-for-the-newer-rpi-4-with-fixed-usb-c-power-circuit

## Pinouts

Model A and B (Original)  | Model A+, B+ and B2
------------- | -------------
<img src="http://elinux.org/images/8/80/Pi-GPIO-header-26-sm.png">  | <img src="http://elinux.org/images/5/5c/Pi-GPIO-header.png">

## install OS image on SD card

- run ```df -h``` to see the devices currently mounted  
  ex of output:  
  ```bash
  Filesystem      Size  Used Avail Use% Mounted on
  # ( .. )
  /dev/mmcblk0p1   32G   32K   32G   1% /media/stephaneag/F0E9-334E
  ```

- the device name 'll be the 'Filesystem' name minus the partition number  
  ex: for '```/dev/mmcblk0p1```' it'll be '```/dev/mmcblk0```', for '```/dev/sdd1```', '```/dev/sdd```'

- to only list the connected SD cards ( if they use the standard naming system ), run:  
  ```df -h | head -1; df -h | grep 'mmcblk\|sdd' | cat```

- unmount any partition related to the SD card using ```unmount /dev/<SC_card_filesys><partition>```  
  ex, for the first partition of '```/dev/mmcblk0p1```':  
  '```unmount /dev/mmcblk0p1```'

- write the .img to the SD card using ```dd bs=4M if=<img> of=/dev/<SC_card_filesys>```  
  ex, for a Raspbian release:  
  ```dd bs=4M if=2015-09-24-raspbian-jessie.img of=/dev/mmcblk0```

- to see some progress about the operation, as ```dd``` doesn't giv any, we can use the following:  
  - to get an update on the progress: ```sudo pkill -USR1 -n -x dd```
  - to get updates on the progress: ```watch -n5 'sudo kill -USR1 $(pgrep ^dd)'```
  - other ways to do so ( ```pv```, ```dcfldd```): <a href="http://askubuntu.com/questions/215505/how-do-you-monitor-the-progress-of-dd">Ask Ubuntu - monitor the progress of dd</a>

## Bypass/replace a lost password
Nb: this should also work for ANY linux system ;)

- plug the SD card containing the RPi system on a laptop, & open the ```/etc/shadow``` file for editing

- in it, find  the line that has the following syntax and correct username:  '```<username>:<hashed_password>:::```'  
  ex of what the line should look like:  
  '```pi:$6$3YkhpIdG$y4C5gTMdnVnhMND9Pe.MFRi3Gi.. ..zTz52bnG1XOcXuhNOpk85xQJfQ.:15710:0:99999:7:::```'

- replace it with either:  
  - another ```<username>:<hashed_password>:::``` line  
  - an empty password for the same user, using ```<username>::::```, ex: ```pi::::```  

- on the next boot of the RPi, no password 'll be asked ( if we set an empty password ) !  
  we could also set a new password if we left it empty, using ```passwd <username>``` as in the following:  
  ```bash
  passwd pi
  <new_password>
  <new_password_again>
  ```
