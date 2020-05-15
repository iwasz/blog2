---
layout: post
title: The any key
permalink: /electronics/the-any-key/
post_id: 318
categories: 
- electronics
---

...which in fact is a one button HID keyboard which you can reprogram to be any key or combination of keys you wish (open source hardware and software). Links for start:*[Device firmware 
**source code**
 + eagle files](https://code.google.com/p/iwasz-sandbox/source/browse/#svn%2Ftrunk%2Ftm4c123-drama-button-v2).

	
* [Host software](https://code.google.com/p/iwasz-sandbox/source/browse/#svn%2Ftrunk%2Fany-key-host-app)              
* **[source code](https://code.google.com/p/iwasz-sandbox/source/browse/#svn%2Ftrunk%2Ftm4c123-drama-button-v2)**
* [Previous attempt.](http://www.iwasz.pl/electronics/a-chaotic-post-on-hid-keyboard-stm32f407-success-stm32f105-fail/)
 
And a quick video (one blurry shoot):

<!-- http://youtu.be/CeLo8d0pDmI -->

At some point, after a few battles I bravely fought with STM32 I wanted to learn something new. I've been a few times on Texas Instrument's site because I wanted to learn more about BeagleBone black and the Sitara CPU that sits on it and spotted the TIVA microcontrolers somewhere on the page. After quick research they looked very promising. It had all I needed that is : can be easily programmed with GCC stack under Linux, has affordable starting platform (they call them launchpads, and they cost $13 and $20 for TM4C123 and TM4C129 respectively) and, what is most important for me, they have well written peripheral libraries and documentation (i.e. at that time I could only rely on opinions from the Web, but after my first project I definitely can confirm that).

[caption id="attachment_322" align="alignright" width="225"]
[![My button assembled](http://www.iwasz.pl/wp-content/uploads/2014/06/IMG_20140603_012851-225x300.jpg)](http://www.iwasz.pl/wp-content/uploads/2014/06/IMG_20140603_012851.jpg) My button assembled[/caption]

So I started a new simple project, which I previously tried to make with STMs and had countless problems with ([here is the link](http://www.iwasz.pl/electronics/a-chaotic-post-on-hid-keyboard-stm32f407-success-stm32f105-fail/)). I've got 
[EK-TM4C123GXL](http://www.ti.com/tool/ek-tm4c123gxl) launchpad and it's great. Somewhere in near future I'll try to write another post which would explain how to start development on Linux with GCC toolchain with this board, but for now I can only assure you that getting started was as easy and quick as one evening (I used my own cross-compiler which is 
[described in previous post here](http://www.iwasz.pl/electronics/toolchain-for-cortex-m4/)). The project aims to construct one button USB-HID keyboard which could be reprogrammed in such a way that pressing the button would send any key code user wishes or even combination of keys if necessary. I imagined, that it would be super cool to have something like that on my desk at work, and if someone comes by and interrupt me with my work, I would ostentatiously hit the big red button which stops the music in my headphones and ask : "what? once again?". TI provides excellent peripheral library with many examples for many evaluation boards. Furthermore they have great USB library which is neatly divided in four tiers dependent on each other. On the very top is the highest level tier called the "Device Class API" which enables one to implement typical devices in just few lines of code (I mean simple HID, DFU, CDC etc.) ST does not have that! Device class API is great, but in fact quite inflexible. For example HID keyboard can have only one interface which is not enough if one wants to implement something more sophisticated. 
[Here are instructions for designing HID](https://www.microsoft.com/china/whdc/archive/w2kbd.mspx) keyboard design with additional multimedia capabilities (which I wanted so bad). Microsoft recommends that there should be at least two USB interfaces in such a device. One should implement a ordinary keyboard compatible with BOOT interface, so such keyboard would be operational during system start up, when there is no OS support yet, and another one would implement the rest of desired keys such as play/pause, volume up, down and so on. I saw quite a few USB interface layouts conforming to this recommendations over the net, including my own keyboard connected to my computer as I write this, so I assume this is the right way to do this. 
[And here is an example of](http://www.usblyzer.com/reports/usb-properties/usb-keyboard.html) USB interface layout as mentioned earlier. HID reports are also provided. So I moved to lower level tiers and it was not so simple. 
[Here you can find all the code that is inside the button](https://code.google.com/p/iwasz-sandbox/source/browse/#svn%2Ftrunk%2Ftm4c123-drama-button-v2). All the magic is done in main.c which could be split in several smaller files, but who cares. Firstly there are USB descriptors. Standard and HID ones:

const tConfigSection *allConfigSections[] = {
        &configSection,
        &interfaceSection1,
        &hidKeyboardSection1,
        &endpointSection1,
        &interfaceSection2,
        &hideKeyboardSection2,
        &endpointSection2
};
Next you have callbacks. My code is heavily based on TI examples, but in some places it is simplified where no advanced functionality is needed. Custom requests are handled in 
**onRequest**
where you can find bits responsible for sending and receiving configuration from the host (using another program running on a PC which is linked below). Configuration (i.e. what key combination should be sent to the host when "any-key" is pressed) is stored in eeprom (functions 
**readEeprom**
 and 
**saveEeprom**
). And of course in 
**main**
 function you can find the main loop with buttons polling and report sending. After connecting the device to a Linux PC it introduces itself as two interface HID device which is silently recognized by Linux (and not so silently by Windows which searches for some drivers for it). What distinguishes this HID keyboard from others is that it recognizes two additional control requests from the host PC which enables user to store and receive combination of keys this device sends when pressed. This requests are prepared in PC application which looks like this: 
[![Any key host app](http://www.iwasz.pl/wp-content/uploads/2014/06/904881400189746306-300x109.png)](http://www.iwasz.pl/wp-content/uploads/2014/06/904881400189746306.png)   Every button on the main screen can be toggled (on the picture above the "play/pause" one is turned on) which immediately sends the configuration data which is stored in eeprom. After closing the host application (which then releases the USB device to the OS) button works as programmed, in situation depicted above behaving as a play/pause keyboard button. Play/pause was my initial intention and I am using it with this function right now, but friend of mine used in on presentation (as arrow down), and also I tested ctrl-alt-del, ctrl-shift-t (eclipse CDT open element), and power among others. Maximum simultaneously pressed keys which can be simulated is 8 for control ones (like ctrl, shift, alt etc) and 6 for regular ones.


[![Any key internals](http://www.iwasz.pl/wp-content/uploads/2014/06/IMG_20140609_234756-300x225.jpg)](http://www.iwasz.pl/wp-content/uploads/2014/06/IMG_20140609_234756.jpg)So there you have it. Feel free to post questions etc. I am also wondering about a "mass production experiment" which would aim to make, say, 10 of those things (with cheaper micro of course!) and sell them on 
[tindie](https://www.tindie.com/) (I have never sold anything made by myself yet). What do you think? Would you buy one of these? What would be reasonable price for this (it is only one button after all... + PC app). I made some very rough calculations and the total cost of one device (assuming production of 100 pcs) would be somewhere around $10, when using MSP430 as a µC and importing casings from China. Not to mention boxes to pack the stuff, soldering (probably in some kind of reflow oven) and sending it all together. So for now it seems overwhelming for me, but who knows.

And for something completely different : what happens when you connect a USB device with VBUS and GND shorted:

Jun  4 08:58:52 diora kernel: [  998.928731] hub 2-1:1.0: over-current condition on port 1
Jun  4 08:58:52 diora kernel: [  999.136827] hub 2-1:1.0: over-current condition on port 2
... and you can hear humming in headphones connected to the PC.

**EDIT**
 : User jancumps on the EEVBlog forums pointed out, that there is an 
[ongoing indiegogo campaign for a similar idea](https://www.indiegogo.com/projects/very-serious-button). Looks quite the same as mine :D


**EDIT2**
 : Dave did a review of the "serious button" 
**this is not mine design, it only looks the same**
:

http://www.youtube.com/watch?v=f2Tr8r2j8wQ&t=12m44s


**EDIT3**
 (09/2014) : 
[Another one on indiegogo with goal of $100k!](https://www.indiegogo.com/projects/bttn/x/7931932)