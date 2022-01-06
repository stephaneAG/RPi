#### Stuff related to getting img/video -> glsl shader runnig in realtime on RPi
- Official demo programs( TO TRY AND STUDY !! ): https://projects.raspberrypi.org/en/projects/demo-programs
- Programs in question: https://github.com/raspberrypi/firmware/tree/master/opt/vc/src/hello_pi
- BIG book on RPi GPU Audio & Video programming ( 448 pages ! ): https://whycan.com/files/members/3/Apress_Raspberry_Pi_GPU_Audio_Video_Programming_148422471X.pdf
- ( my guess ) an adapted version of the gello_triangle program to NOT draw on framebuffer but file instead: https://github.com/matusnovak/rpi-opengl-without-x 
- R Adafruit's Video Looper project's hello_video: https://learn.adafruit.com/raspberry-pi-video-looper/hello-video
- example DispmanX code possibly interesting: https://forums.raspberrypi.com/viewtopic.php?t=213964
- some quick infos on OpenMax: https://en.wikipedia.org/wiki/OpenMAX
- pishadertoy: https://forums.raspberrypi.com/viewtopic.php?t=10246
- pishadertoy repo: https://github.com/dff180/pishadertoy
- quick shadertoy oldschool blob demo: https://www.shadertoy.com/view/MdXGDH
- GLSL viewer ( seems to suport img/video in & also a LKGF-specific param ? ): https://github.com/patriciogonzalezvivo/glslViewer/wiki/Audio-and-Video-Textures
- 'old' tool by the creator of the RGB LEDs coube that uses an OpenGL shader to drive them ( + mapping ): https://github.com/polyfloyd/shady
- replacement player for omx ?: https://github.com/Hemisphere-Project/HPlayer
- OMXPlayer: https://github.com/popcornmix/omxplayer
- pi3d tool :https://pi3d.github.io/html/FAQ.html
- using glsl hacker ( which was deprecated in favor of GeexLab ): https://www.geeks3d.com/20150611/simple-video-player-for-raspberry-pi/
- geeXLab: https://geeks3d.com/geexlab/tutorials/
- using oxPlayer stg: https://forum.openframeworks.cc/t/doing-color-correction-on-video-in-real-rime-on-a-raspi/30154/3
- adding glsl support to ffmpeg: https://github.com/numberwolf/FFmpeg-Plus-OpenGL/blob/main/README_EN.MD
- interesting 'zero copy' hdmi to csi input (HDMI video input to OpenGL texture PI 4): https://forums.raspberrypi.com/viewtopic.php?t=281296

#### Compiling stuff from Unity or Unreal Engine to the RPi
- https://medium.com/geekculture/run-unity-game-on-raspberry-pi-4-with-picade-arcade-machine-c54210d64b7a
- quote author 'don't xpect perfs': https://github.com/rdeioris/UnrealOnRPI4

#### Driving specific displays with specific Rpi models
- RPi0 capable of 2k or more ( lessens the fps ): https://forums.raspberrypi.com/viewtopic.php?t=186820

#### Adding external peripherals on gpios & related Device Tree drivers:
- Adding Ethernet port via gpios ( hugely simpler than USB ;p ): https://www.raspberrypi-spy.co.uk/2020/05/adding-ethernet-to-a-pi-zero/
- RPi A, so my code to force the mode was good yet I was using the 'wrong' USB port ? ( also means MAX3421 needed as well for the RPi A models: https://www.journaldulapin.com/2019/04/30/raspberry-pi-a-otg/
- 'quite old' hints on how to drive gpios using a DT overlay ( no possible using config.txt ): https://raspberrypi.stackexchange.com/questions/43825/how-do-i-make-a-device-tree-overlay-for-just-a-single-gpio
- the now available way to do so: https://www.kernel.org/doc/Documentation/devicetree/bindings/gpio/gpio.txt
- interesting way to power a RPi 0 ( USB to ethernet + A/C adaptor bundle: https://www.journaldulapin.com/2016/01/21/raspberry-pi-zero-google/

#### Source that helped me to implement the max3421 USB Host Shield Mini ( the overly & driver part )
##### R: additionally to those mentionned in my tmp writeup at https://github.com/stephaneAG/RPi/blob/master/max3421_spiUsbHost.md
- Official RPi doc, TO TAKE .md 'quick' notes from !! ( at least, on the kernel & dt stuff for now ): https://www.raspberrypi.com/documentation/computers/getting-started.html
- DT forums on rpi.com ( I guess where to post my fix & questions for the max3421 ): https://forums.raspberrypi.com/viewforum.php?f=107
- Thomas Pettazzoni's example on bootlin for the 'correct' way to do so: https://bootlin.com/blog/enabling-new-hardware-on-raspberry-pi-with-device-tree-overlays/
- The ONE stack overflow that helped me made it possible: https://raspberrypi.stackexchange.com/questions/39845/how-compile-a-loadable-kernel-module-without-recompiling-kernel
- The OTHER one that helped ( digging & looking for different file paths with same logic ): https://jumpnowtek.com/rpi/Using-mcp3008-ADCs-with-Raspberry-Pis.html
- MAX3421, digging the drivers souce code to better understand its inner workings: https://github.com/raspberrypi/linux/blob/rpi-5.10.y/drivers/usb/host/max3421-hcd.c#L1647
- MAX3421, GPOUTx seems supported, but no clues yet if supporting multiple GPOUTx to be set via the DT overlay & possibly later: https://github.com/raspberrypi/linux/blob/rpi-5.10.y/drivers/usb/host/max3421-hcd.c#L1653
- possibly interesting comms to digg 'Raspberry Pi Zero OTG Mode': https://gist.github.com/gbaman/50b6cca61dd1c3f88f41
- related overlay ( very interestingly supports different SPI configurations ): https://github.com/raspberrypi/linux/blob/rpi-4.4.y/arch/arm/boot/dts/overlays/mcp3008-overlay.dts
- 'extras' good read for long night without sleep: https://github.com/raspberrypi/firmware/blob/master/extra/dt-blob.dts
- post to read/digg ( Loading of kernel module with overlay ): https://forums.raspberrypi.com/viewtopic.php?t=214202
- interesting read as well on the boot order & steps: https://raspberrypi.stackexchange.com/questions/41993/module-loads-with-modprobe-but-doesnt-in-boot
- bcm283x: https://github.com/raspberrypi/linux/blob/rpi-5.10.y/arch/arm/boot/dts/bcm283x.dtsi
- few infos on RPi SPI: https://elinux.org/RPi_SPI
- RPi overlays: https://github.com/raspberrypi/firmware/tree/master/boot/overlays
- RPi overlays readme: https://raw.githubusercontent.com/raspberrypi/firmware/master/boot/overlays/README
- Rpi dts: https://github.com/raspberrypi/linux/tree/rpi-5.10.y/arch/arm/boot/dts
- https://www.kernel.org/doc/Documentation/devicetree/bindings/arm/bcm/bcm2835.yaml
- 

#### Thinking about Nico's stuff
- AirSim ( which also does automotive stuff using the Unreal Engine ): https://microsoft.github.io/AirSim/
- some not that good looking framework on unity this time: https://www.youtube.com/watch?v=EaY5QiZwSP4&t=167s
