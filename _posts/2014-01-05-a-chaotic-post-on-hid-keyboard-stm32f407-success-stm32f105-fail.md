---
layout: post
title: A chaotic post on HID keyboard. STM32F407 success, STM32F105 fail
permalink: /electronics/a-chaotic-post-on-hid-keyboard-stm32f407-success-stm32f105-fail/
post_id: 249
categories: 
- electronics
---
This is a quick dev-log post on my latest design, which was only partially successful. I have STM32F407-DISCOVERY board on which I successfully implemented a HID keyboard with only one keyboard. At first it reported that 'a' key was pressed every time user pressed the blue button, then, according to my plan I changed this to play/pause button, which can turn music on and of. It works under Linux and Windows (only 'a' version tested under win though). Then I decided to make a board for this and, since F407 is quite expensive, in fact too expensive for simple one-key keyboard, I decided to use something simpler. The slowest and cheapest µcros that support STM32_USB-Host-Device_Lib are those labeled as "connectivity line" i.e. STM32F105 and STM32F107. I've got myself two STM32F105R8T6 and made a board, which fits into a case I also bought. The case is labeled as "XB2-ES542"  :

![The any key PCB](/assets/board-300x129.png)

![The USB button](/assets/3729255047-300x168.jpg)

Eagle schematic and board are [here](https://code.google.com/p/iwasz-sandbox/source/browse/#svn%2Ftrunk%2Fstm32f105-drama-button%2Fboard), OSH Park shared projest is [here](http://oshpark.com/orders/new/ZLB4alej). I assumed (incorrectly) that porting my program from F407 dev board to my custom board featuring different micro will be easy since they are quite similar. I was wrong. And I don't have a dev board for F105 nor 107. Ok, but first things first. As I mentioned, program works on F407, so let me write down some random thoughts which emerged during the process of making this work:

## Few facts about HID devices (that I learned)

All data exchanged resides in structures called reports. The host sends and receives data by sending and requesting reports in control or interrupt transfers. The report format is flexible and can handle just about any type of data, but each defined report has a fixed size. The device’s descriptors must include an interface descriptor that specifies the HID class, a HID descriptor, and an interrupt IN endpoint descriptor. The firmware must also contain a report descriptor that contains information about the contents of a HID’s reports.

So there are two additional descriptors when comparing to the 'vendor specific' device I made recently (there may be third, optional descriptor as well). First is HID class device descriptor and it specifies which other class descriptors are present (for example report descriptors or physical descriptors).

A HID can support one or more reports. The report descriptor specifies the size and contents of the data which this device generates. Physical descriptors at the other hand are optional pieces of data which describe the part(s) of human body used to operate the HID device. The HID class does not use subclasses to define most protocols. Instead, a HID class device identifies its data protocol and the type of data provided within its Report descriptor.


[Here](http://www.usb.org/developers/devclass_docs/Hut1_11.pdf) on page 53 you can find all key codes defined by the HID spec. Document "Device Class Definition for Human Interface Devices (HID) Version 1.11" on page 62 has very useful information regarding keyboard implementation. Especially crucial are those bits about when to send a data report: "The keyboard must send data reports at the Idle rate or when receiving a Get_Report request, even when there are no new key events." I mixed up the rates and my HID keyboard acted unpredictable. Only after adjusting wait period for 4ms * idle rate things went OK.


Then I started to getting familiar with the report descriptors, but nah, the more I read HID specification the more I realized this is too much complexity, and too much effort than I wanted to put into this project. At first I was like, "OK let's read the whole spec, it has 97 pages, I've read longer specs before, not a problem". But hey, this was meant to be a simple, few evenings project, and this HID spec turned out to be surprisingly complex (I mean report descriptors in particular. When I came to Push and Pop items I refused to read further). The better and simpler way of accomplishing this project was to grab some descriptors from the net, and so I did:

* [Here I found useful](http://www.obdev.at/products/vusb/hidkeys.html)report descriptor for regular keyboards (like qwerty ones). Other, special buttons are implemented by other means (other items in reports) as I noticed before.
* [Here are some interesting](http://www.tretec.it/public/webroot/media/PSE/090224_USB_HID.pdf)report descriptors which looks like something I want to do (multimedia control). Seems to me, that all those volume, play/pause and other knobs are implenmented in some other means that regular keyboards are. There are different type of "usage" used i.e. regular keyboard has 0x09 0x06 (USAGE (Keyboard)), but in above document there is 0x09 0x01 (Usage (Consumer Control)) used.

There is also a tool on USB-IF page, which helps to assembly HID report descriptors, and I can confirm that it runs on Wine, but that is all I can say about it. I suppose you still have to know the specs, and know what you are doing while using this thing. Last but not least the source code which runs flawlessly on STM32F407-DISCO:
* [The source code for this project (works OK on STM32F407).](https://code.google.com/p/iwasz-sandbox/source/browse/#svn%2Ftrunk%2Fstm32f407-drama-button)

## Failed attempt to port it to STM32F105

Then I started to move to my custom board depicted above, which I didn't managed to accomplish. Actual status of the whole device (source code linked below, Eagle files above) is that, after connecting the USB, device initializes itself (i.e. USB stack gets initialized) then it gets quite a few reset requests (like 10) and then it hangs. Wireshark + usbmon shows "Malformed packet" when device tries to send the Device descriptor to host. Random notes from development:

* Note : STM32F105 and 107 are called "connectivity line microprocessors". It is useful to know that since there are many resources for STM32F1x out there which are tagged like "value line", "connectivity line" etc.
* I copied my previous project [stm32f407-drama-button](https://code.google.com/p/iwasz-sandbox/source/browse/#svn%2Ftrunk%2Fstm32f407-drama-button) into new place in my SVN repository : [stm32f105-drama-button](https://code.google.com/p/iwasz-sandbox/source/browse/#svn%2Ftrunk%2Fstm32f105-drama-button).
* I downloaded and unpacked STM32F10x_StdPeriph_Lib_V3.5.0 library from 
[here](http://www.st.com/web/catalog/tools/FM147/CL1794/SC961/SS1743/PF257890). Main page for this micro is 
[here](http://www.st.com/web/catalog/mmc/FM141/SC1169/SS1031/LN1564). Current version of standard peripheral library for STM32F10x as of writing this is 3.5.0.
* I replaced `/STM32F4xx_StdPeriph_Driver` with `/STM32F10x_StdPeriph_Driver`.
* Replaced `CMSIS` folder. Removed `Docs` folder.
* Made new toolchain with crosstool-ng fine tuned for Cortex-M3 µC.
* Copied and modified stm32f105-crosstool.cmake.	
* StdPeriph comes with ld-scripts for various dev-boards. I figuret out that :
  * STM3210B-EVAL has STM32F103VBT6 µC.	
  * [STM3210C-EVAL](http://www.st.com/st-web-ui/static/active/en/resource/technical/document/user_manual/CD00212441.pdf) has STM32F107VCT µC.	
  * [STM3210E-EVAL](http://www.st.com/st-web-ui/static/active/en/resource/technical/document/user_manual/CD00178166.pdf) has STM32F103ZGT6 µC.
* So linker script for STM3210C-EVAL is best suited for me and will check it first. Have it copied and modified. In "drama button" project I use STM32F105R8T6 which has:
  * 64kB of flash,	
  * 64kB of RAM
* I copied `stm32f10x_conf.h` from examples into src. CMSIS uses it somehow and I think this is bad design. Lower level library depends on higher level header file?
* Had trouble when used GCC-4.8.0 (linaro version made with ct-ng 1.19). Works fine with GCC-4.7.0 (linaro version prepared with ct-ng 1.18). Some strange assembler errors when compiling `core_cm3.c` file poped out.  [Here](http://www.embedds.com/st32mvldiscovery-project-template-for-gcc/#comment-54664) a guy in comments had similar issue, and someone told him to download fresh CMSIS library, because this provided with StdPeriph is old. I can believe that, because for example ST USB OTG library comes with StdPeriph bundled inside and it is also some old version. So my rule of thumb now is to collect newest versions of all the individual libraries, even when they are distributed together (i.e. OTG library virtually has everything required to compile the examples it provides, but now I throw it away and get fresh ones).

**EDIT** : when compiling with gcc-4.7.0 with heavy optimizations (-O3) the same assembler error emerges. Message is

```
/tmp/ccOfylWN.s: Assembler messages:
/tmp/ccOfylWN.s:646: Error: registers may not be the same -- `strexb r0,r0,[r1]'
/tmp/ccOfylWN.s:675: Error: registers may not be the same -- `strexh r0,r0,[r1]'
```

I downloaded and upgraded CMSIS from here : [https://silver.arm.com/browse/CMSIS#](https://silver.arm.com/browse/CMSIS#) (registration required). Upgraded from version 1.3.0 to 3.2.0. No *.c file this time, only header files.

When ported from STM32F407-DISCO to my custom STM32F105 board, program refused to operate. Kernel log says that:

```
Jan  3 00:08:09 diora kernel: [ 4993.895502] usb 7-2: new full-speed USB device number 4 using uhci_hcd
Jan  3 00:08:09 diora kernel: [ 4993.959542] hub 7-0:1.0: unable to enumerate USB device on port 2
```

And oscilloscope says, that... well... seems OK to me (compared for example to Wikipedia article on USB) :)

![Oscilloscoe1](/assets/map001-300x180.png)
![Oscilloscoe1](/assets/map002-300x180.png)

Debugger shows that program is running, It correctly invoke Reset_Handler, and then hits the main function.

Mistakes that I know I made:

* I made a circuit with µC that I don't have development board for (Assumed, that porting tested program from STMF407 to STMF105 will be easy. Wrong).	
* I screwed up a SWD port. Issues with signal integrity (probably due to lack of termination resistors?). This made debugging with GDB harder. It would hang, or disconnect at random points.	
* I forgot to add serial console output for STDOUT. Terrible mistake. I've wired that up, but board looks messy now.	
* Simple delay functions was dependent on optimization switches.
And then, after about 10 evenings / nights I gave up and put this project on the shelf for some time.