---
layout: post
title: STM32F042 option bytes under Linux
permalink: /electronics/2020-07-29-stm32f042-option-bytes-linux/
categories: 
- electronics
---
These are instructions that worked for me when faced with a problem of modifying option bytes in stm32f042. The thing I was trying to do was to turn off BOOT0 pin support (PB8 in my chip) which then would eliminate the need to pull it. Then I could use this pin for something else, for example as a `CANbus` `canRx` input.

A bunch of useful links:
* [Stack overflow question](https://stackoverflow.com/questions/48927028/openocd-how-to-write-option-bytes-to-stm32f4) which put me on the right track.
* [Openocd manual](http://openocd.org/doc/html/General-Commands.html)
* Reference manual of whatever chip you are using. The chapters of interest are entitled:
  * 3.2.2 Flash program and erase operation 
    * Unlocking the Flash memory
    * Option byte programming
  * 4 Option byte 
  * A.2.5 Option byte unlocking sequence code example
  * A.2.6 Option byte programming sequence code example
  * A.2.7 Option byte erasing sequence code example

# Preparation
Turn the power on. Run the `openocd` (mind the paths):

```sh
openocd -f /home/iwasz/local/share/openocd/scripts/interface/stlink.cfg -f /home/iwasz/local/share/openocd/scripts/target/stm32f0x.cfg 
```

Connect to the running `openocd` to get access to the interactive shell:

```sh
telnet localhost 4444
```

Verify that the **FLASH_CR.LOCK** is set, meaning that the entire flash is locked. Flash base address is : 0x40022000 which tells that **FLASH_OPTKEYR** address is 0x40022008 and **FLASH_CR** address is 0x40022010. Read FLASH_CR as so (00000080 means the **LOCK** flag is set):

```sh
mdw 0x40022010
0x40022010: 00000080 
```

# Unlock the flash and option bytes
Unlock the flash directly performing the unlocking sequence on the **FLASH_KEYR** (0x40022004) register. The procedure is described in the Reference Manual in the chapter *Flash program and erase operation*, and also in the examples section as a source code snippet. You simply write two values to the same **FLASH_KEYR** register:

```
mww 0x40022004 0x45670123
mww 0x40022004 0xCDEF89AB
```

Reassure that the flash is unlocked. Read the **FLASH_CR**:

```
mdw 0x40022010
0x40022010: 00000000 
```

This means that flash is unlocked, and **LOCK** is 0 (and all other flags are 0 as well).
 
Unlock option bytes for writing (which is described in the reference manual as *setting the OPTWRE bit in the FLASH_CR*). The sequence is basically the same as in the previous step, with the only difference that this time we write the magic unlocking values to **FLASH_OPTKEYR** (0x40022008):

```
mww 0x40022008 0x45670123
mww 0x40022008 0xCDEF89AB
mdw 0x40022010           
0x40022010: 00000200 
```

Now instead `00000000` we see `00000200` which means that **OPTWRE** flag is set.

# Clear the option bytes
Clear option bytes (they are stored in the FLASH after all). This is done by first setting **FLASH_CR.OPTER** (erase), and then **FLASH_CR.STRT** (start operation). Note that **FLASH_CR.OPTWRE** must be still set:

```
mww 0x40022010 0x00000220
mww 0x40022010 0x00000260
```

All (?) option bytes should now read 0xffffffff.

# Modify the option byte
Enable programming by setting **FLASH_CR.OPTPG**:

```
mww 0x40022010 0x00000210
mdw 0x40022010
0x40022010: 00000210
```

Store a value (only half-words). In this example, I am modifying the *User and read protection option byte* (0x1FFFF800). In particular, I'm setting the **BOOT_SEL** flag.

What is described as a *User and read protection option byte* is in fact a 32bit word containing the user option byte [23:16] (and its "complement"), and the RDP option byte [7:0] (and its "complement").

So wanting to change the **BOOT_SEL** we calculate, that the proper value of the user option byte [23:16] is 0x7f, and thus its "complement" byte (a negation) should be 0x80. 

Also, I wanted to restore the RDP to its factory value, because it got lost for some reason.

```sh
# Set read protection option byte to 0xaa which denotes level 0 (the default) 
mwh 0x1ffff800 0x55AA
# Set BOOT_SEL to 1 to ignore BOOT0 pin. 
mwh 0x1ffff802 0x807f
# Verify
mdw 0x1ffff800
```
