---
layout: post
title: Internet thermal printer part 2
permalink: /electronics/internet-thermal-printer-part-2/
post_id: 396
categories: 
- electronics
---

The printer I bought and 
[described in the previous](http://www.iwasz.pl/electronics/internet-thermal-printer/) post really disappointed me. I didn't spend some huge amount of time on that (say 3-4 evenings), but I dig into the subject so deep so I couldn't help myself but do some more hacking. First of all I wanted to know if my cool looking, but quite useless printer can be used in some other way (i.e. the printing head) and whether that is the main board which is broken or the head itself. If the former was true, and the head was OK, I would try to communicate with the head and thus make something which would pretty much implement the whole printer main board that was broken. But if the head was broken, I couldn't do anything but to abandon the project or find another printer. And it's funny, because, as you may have seen at the end of my previous post, this is exactly I've written not to do. But I just love it. When the work you do all day every day is stupid and pointless, when you are constantly bothered with more and more irrelevant things, and after all day you are tired and discouraged, what would you do after arriving home (excluding household duties :D)? Grab a beer, sit and watch TV? Hell no! Grab a beer and tinker some more! It calms me down you know (unless I'm stuck for to long). The printer head used in my 
**Intermec PW40**
 is a Seiko (
[www.sii.co.jp](http://www.sii.co.jp)) 
[**SII 
LTP3445**
(datasheet here)](http://zival.ru/sites/default/files/download/ltp3445.pdf) and it is obsolete. New designs are encouraged to use LTPV445.

So what I did is that I soldered a bunch of wires to the main board to be able to speak directly to the printing head. The resulting wiring looks like that:

![Bunch of wires soldered](/assets/IMG_20140708_010418-225x300.jpg)

Then I grabbed signals with a logic analyzer and an oscilloscope to figure out what is malfunctioning (i.e. when the printer was operating). In my opinion, the main board is broken, because printing short strings like 'A\r\n' works OK, and all signals seems to be correct (i.e. 832 bits per row are transferred and quite a few rows are present). But when longer strings are submitted, the whole transmission appears to be corrupted at some point. Serial data burst is clearly shorter, like interrupted. Unfortunately I have made a screenshot of correct transmission (A\r\n), and don't have the corrupted one now (and the board is not operational since I removed the FFC socket). Here's the screen:

![Transmission dump](/assets/screenshot-300x183.png)

Above : a correct transmission to the thermal head issued by the original Intermec PW40 main board. Letter 'A' is being transmitted.

The next step was to wire up some circuitry to actually drive the head while it was still soldered to the original main board. I didn't want to break it then, but later it ceased to be a priority :D My setup consists of:

* [Texas Instruments EK-TM4C123GXL](http://www.ti.com/tool/ek-tm4c123gxl) as the main logic unit.	
* [DRV8835 Dual Motor Driver Carrier](http://www.pololu.com/product/2135) from Pololu. It uses a 
[DRV8835](http://www.ti.com/product/drv8835) chip from Texas Instruments.	
* Two 
[CD40109B](http://www.ti.com/product/cd40109b) level shifters from Texas Instruments. Breadboard looks like this:

![Bread board](/assets/IMG_20140708_000707-225x300.jpg)

Above : the circuit. You can see that the printer is more or less intact i.e. the head is mounted on the main board and the plastic frame. Later on I decided to disconnect the head from the original main board.

Shifters are controlled with 3.3V and output 5V for the head's logic. The whole contraption is powered from a laboratory power supply which was set to 5V with low current limit to prevent smoke and fire in case of errors in wiring on my side. The setup drawn about 0.1A when idle and 2.5A when feeding the paper. Driving the motor was pretty easy, I did stepper motor before, so I rather quickly caught up with this one. But the head took more time and at some (thankfully short) time I was stuck. First, the DST signal (DST is for power and thus temperature) circuitry on the main board is secured with some (I believe) TTL logic. The idea is that if the thermistor says to the µC that he head is overheating, the µC shuts the head down. This is the first protection mechanism programmed in software (BTW manual says that if overheat, the head may cause skin burns, smoke and even fire. It is a thermal one after all). But there is another protection mechanism done in hardware which shuts down the head if the software one malfunctions. I believe, that the two mechanisms are AND-ed by some TTLs. The protection mechanisms are pulling down the DST in case of trouble. In my case, when actually two logic circuits were connected to the head, this situation caused problems, because the original main board, which was not powered, pulled the DST down all time. The solution to this was to cut the trace and that was it (if not cut, the DST would stay low no mater what level I was trying to drive it. Oscilloscope shown only 100mV level changes, obviously to small to be useful).

![Logic analyzer window](/assets/screenshot1-300x183.png)

Above : a transmission. A 12*8 pixel wide bar strip. 12 x 0xaa.

But still no luck after the DST problem was resolved, so I decided that something else on the original main board is interfering and I need to disconnect the head from it in sake of connecting to it directly. Didn't have spare FFC socket though (Molex 25 pin 1.25 mm pitch rare and obsolete), so after obtaining a wrong one from Farnell (bought 1 mm pitch instead od 1.25 duh!) I soldered the wires directly to the FFC cable. Looks awful, but is rigid:

![Wires soldered to the FFC cable](/assets/IMG_20140711_233318-300x225.jpg)

Still no luck! What the hell! Logic analyzer still happily shows correct bursts of data, so for the third time rewired the breadboard and checked levels with an oscilloscope. And curious thing revealed : all levels (shifted up) were 0-4V instead od 0-5V. I have completely no idea why? My power supply is a cheap one, but can 1 or 2 amps of load cause 1V drop? Must investigate further. 

**EDIT**
 my cheap counterfeit Saleae logic analyzer must have somewhat low input impedance and it was it that caused significant voltage drop on logic signals. Disappointing. On the picture below you can see (far left) that only after increasing the voltage repeatedly, the printer started to print:

![A printout](/assets/ltp3445-first-success-1024x215.jpg)

I'm excited!
