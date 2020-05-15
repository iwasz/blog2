---
layout: post
title: BlueNRG + STM32F7-DISCO tests
permalink: /uncategorized/bluenrg-stm32f7-disco-tests/
post_id: 458
categories: 
- Uncategorized
---

* Tested 32bit cross-compiler from Launchpad, it worked OK, but binaries were much bigger, and debugger lacked Python support which is required by QtCreator which is an IDE of my choice (for all C/C++ development including embedded).

![Discovery board](/blog/assets/IMG_20160321_020856-300x225.jpg)
	
* Made my own compiler with crosstool-NG (master from GIT). Used my own tutorial, which can be found 
[somewhere over this blog](http://www.iwasz.pl/electronics/toolchain-for-cortex-m4/).	
* Verified that a "hello world" / "blinky" program works. It did not, because I copied code for STM32F4, and I messed the clock configuration. Examples from STM32F7-cube helped.	
* I [ported my project](https://github.com/iwasz/blue-nrg-test) which aimed to bring 
[my BlueNRG dev board](http://www.st.com/web/catalog/tools/FM116/SC1075/PF260517) to life. The project consists of a USB vendor-specific class for debugging (somewhat more advanced, and tuned for the purpose than simple CDC class), slightly adapted BlueNRG 
[example code from ST (for Nucleo)](http://www.st.com/web/en/catalog/tools/PF261442), and some glue code.	
* I switched to STM32F7-disco because I couldn't get BlueNRG to work with STM32F4-disco. F7 has Arduino connector on the back, so I thought it'd be better than bunch of loosely connected wires which was required for F4. At that time I thought that there is something wrong with my wiring.	
* On STM32F7-disco BLE still would not work. I hooked up a logic analyzer, and noticed that MISO and CLK is silent. It turned up that I have to use an alternative CLK configuration on BlueNRG board which required to move 0R resistor from R10 to R11. Although it was small (0402), at first I had a problem desoldering it, probably because of the unleaded solder used. See image.

![Jumper configuration changed](/blog/assets/IMG_20160320_175826-300x182.jpg)

* Next problem I had, was with SPI_CS pin. According to the BlueNRG-dev board docs, I wanted to use D8 pin (this is PI2 pin on STM32F7-disco), but it didn't work. Glimpse on the schematics revealed that the SPI_CS is connected to A1, which is marked as "alternative" in the docs. So IMHO this is an error in the docs.
* Final pin configuration that worked was:

``` cpp
// SPI Reset Pin
#define BNRG_SPI_RESET_PIN GPIO_PIN_3
#define BNRG_SPI_RESET_PORT GPIOI
// SCLK
#define BNRG_SPI_SCLK_PIN GPIO_PIN_1
#define BNRG_SPI_SCLK_PORT GPIOI
// MISO (Master Input Slave Output)
#define BNRG_SPI_MISO_PIN GPIO_PIN_14
#define BNRG_SPI_MISO_PORT GPIOB
// MOSI (Master Output Slave Input)
#define BNRG_SPI_MOSI_PIN GPIO_PIN_15
// NSS/CSN/CS
#define BNRG_SPI_CS_PIN GPIO_PIN_10
#define BNRG_SPI_CS_PORT GPIOF
// IRQ
#define BNRG_SPI_IRQ_PIN GPIO_PIN_0
// !!!
#define BNRG_SPI_IRQ_PULL GPIO_PULLDOWN
#define BNRG_SPI_IRQ_PORT GPIOA
```

* Next thing that got me puzzled for quite a time was that after initial version query command, which seemed to work OK (I got some reasonably looking responses) the communication hanged. IRQ pin went high, and wouldn't go low. So µC thought that there is data to read and was continuously reading and reading.	
* Only after I read in 
[UM1755 User manual](http://www.st.com/st-web-ui/static/active/en/resource/technical/document/user_manual/DM00114498.pdf) that IRQ goes HI when there is data from NRG to µC, and it goes HI-Z when there is no data, I double checked the schematics, and found that BlueNRG-dev board has no pull-down resistor in it. The ST example code I had also hadn't have IRQ pin configured in pull-down state, so I wonder how this could work. Maybe Nucleo boards have pull-down resistor soldered there.	
* For now (after 3 or 4 nights, and I consider myself as quite experienced) I got this (see photo):
  
![Linux is seeing the BlueNRG](/blog/assets/Screenshot-from-2016-03-21-02-09-30-300x217.png)

	
* Quick update made the next day morning. My Linux computer is able to connect to the test project, and query it (right now it simply counts up):

![Some console output](/blog/assets/Screenshot-from-2016-03-21-13-05-25-300x179.png)
