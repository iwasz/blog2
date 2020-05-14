---
layout: post
title: Internet thermal printer
permalink: http://www.iwasz.pl/electronics/internet-thermal-printer/index.html
post_id: 345
categories: 
- electronics
---

The idea is shamelessly 
**stolen**
 from 
[this hackaday.io project](https://hackaday.io/project/1085-Pigeon-Post). 
**EDIT**
 : It evolved... What this project is intended to be:

*A toy printer (for my son) with some light and sound signal connected to the Internet and accessible by some web interface. Anyone with password (basic auth configured in .htaccess) could send a graphic and/or text message to the printer which would immediately flash the light, beep the buzzer as well as print the message. Protocol to the printer (on the network level) : whatever a.k.a my own & super-simple.

What this project shall not become 
**EDIT**
 : It evolved.. (note to myself because I tend to complicate things):

*An over-engineered wireless CUPS compatible Postscript full featured printer which also makes coffee.
After deciding that I would try to make such a thing which took approx. 1 second after seeing Jim's site I went to 
[allegro.pl](http://allegro.pl) (local EBay. BTW we have ebay.com.pl here in Poland, but allegro seems to be winning the battle) and found something printer-ish alike and seemingly broken, with some parts missing. It is a 
**Intermec PW40**
 mobile printer. Useful links I found on this printer:

*[Manuals - Intermec site](http://www.intermec.com/support/manuals/search.aspx?categoryid=7&familyid=11&productnodeid=PRTRPW50A) (those are for PW50, but I assume they are compatible in some way).

	
*[Intermec](http://community.intermec.com/) community - they even have forum, and some community around the site.
Photos after dismantling the thing:

[gallery ids="378,375,376,377,379,380,381,382,383,384"]

Looks like it 
[uses ESC/P](http://en.wikipedia.org/wiki/ESC/P) like Jim's printer and 7.2V battery pack also. Looks promising (at least some standard language). Elements found on the main board of PW-40:

*[Toshiba TMP95C061BF](http://pl.mouser.com/ProductDetail/Toshiba/TMP95C061BFZ/?qs=f0GatGUIxV3RUG14ubT6RQ==) : some 16 bit microcontroller. 
[TLCS-900](http://en.wikipedia.org/wiki/Toshiba_TLCS) family. No free C compiler available according to Wikipedia.

	
*NEC 
[D431000AGW](http://search.datasheetcatalog.net/key/D431000) : SRAM 1Mb.

	
*[ST M29F040B](http://pdf.datasheetcatalog.com/datasheet/SGSThomsonMicroelectronics/mXyxzsz.pdf) : Flash 4Mb.

	
*LinearTechnology 
[LTC1384CG](http://www.51samples.com/upfiles/Manual/1384fa%5B1%5D.pdf) : this is what I was looking for : a RS232 transceiver.

	
*[CS8130](http://www.cirrus.com/en/pubs/proDatasheet/CS8130_F1.pdf) : IR transceiver.
I've written the LTC chip looks promising, because it connects the printer to the outside world, and gives a hint where to start hacking. It translates RS232 high voltage levels to TTL, but since I wanted to drive the printer directly from some µC I needed to bypass the LTC. After some research I determined what follows : RS 232 port (this with RJ socket) is connected to pins 14 (232 input) and 15 (232 output). Corresponding TTL pins are : pin 13 (logic output), and 12 (logic input). So as far as I am reasoning correctly :

*Pin 13 is connected to the Toshiba's RX pin.

	
*Pin 12 is connected to the Toshiba's TX pin.

	
*Whole device can be powered from 12V supply (I read that somewhere).

	
*Let's try it! Seems to work. At least PC and the printer are communicating. Wiring looks like this:
[gallery ids="386,385"]
Costs so far:

*Printer : 25PLN ($8).

	
*10 rolls of thermal paper 20PLN ($7)

[Intermec provides a CUPS driver](http://www.intermec.com/public-files/technology-briefs/en/CUPS-Printing-In-Linux-UNIX-For-Intermec-Printers-Tech-Brief.pdf) for Linux which enables you to use their printer as regular printer in the OS. Apparently PW40 isn't supported. I successfully compiled and installed the software, but printing a random text file gave me some gibberish. After that I tried to communicate with te printer in ESC/P language directly, but with no luck. I 
[described my problems on the Intermec](http://community.intermec.com/t5/General-Intermec-Device/PW40-is-it-broken-or-not/m-p/27118) forums and still waiting for some reply. In short the problem is, that I don't really know for sure if this is me doing something wrong, or the printer is broken (it was sold on auction as broken, but seller couldn't tell for sure if it is really broken or not). So after two evenings the situation looks that I am able to print only one character in a row. If I'm sending more than 1 character to print, it hangs. To make matters worse, my printer won't print a self test page as it is described in the manual. It feeds paper a little and that's all. At the other hand I 
[found a datasheet](http://zival.ru/sites/default/files/download/ltp3445.pdf) of the printer head used in my printer, but using it directly would be a triumph of form over the content I'm afraid, and I don't have enough time for that (i.e. making my own printer from scratch). But I'm overambitious you know, so who knows...

[caption id="attachment_387" align="alignnone" width="300"]
[![This is the only thing It can print. If I try to print more than 1 character in a row, It hangs.](http://www.iwasz.pl/wp-content/uploads/2014/07/IMG_20140620_123905-300x225.jpg)](http://www.iwasz.pl/wp-content/uploads/2014/07/IMG_20140620_123905.jpg) This is the only thing It can print. If I try to print more than 1 character in a row, It hangs.[/caption]