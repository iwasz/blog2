---
layout: post
title: BlueNRG-2-Android. Source code troubleshooting, bonding, and privacy
permalink: /electronics/bluenrg-2-android-source-code-troubleshooting-bonding-and-privacy/
post_id: 518
categories: 
- electronics
---

The objective of the firmware presented in this article is to provide an example which would implement the following functionality:

* After resetting the BlueNRG-2 (called the "device" or simply BlueNRG later on) it would make itself general discoverable and accept a connection from whatever connects first.
* It would bond with this first connecting thing (called the "cellphone" later on) and allow it to connect and modify GATT characteristics exclusively.	
* No other central device (cellphone) would be allowed to modify device's characteristics (or even connect).
* This BlueNRG device would work with modern iOS and Android phones (it's 2018 when I'm writing this).
  
My starting point was an example from 
[STSW-BLUENRG1-DK](https://www.st.com/content/st_com/en/products/embedded-software/evaluation-tool-software/stsw-bluenrg1-dk.html) version 3.0.0 named `BLE_Security` which resides in 
`BlueNRG-1_2-DK-3.0.0/Project/BLE_Examples/BLE_Security/src/peripheral` directory. This package comes as an executable if I remember correctly, but most of the ST's execs run under wine (wine-3.6 (Ubuntu 3.6-1), Ubuntu version 18.04). So just after incorporating the example source code into my project, I was able to pair, bond, connect and read characteristics on the BlueNRG-2 using 
[nRF connect app](https://play.google.com/store/apps/details?id=no.nordicsemi.android.mcp) running on Android 8 on Huawei Honor 9. The problem arisen though, when I tried to disconnect, and then connect again. In such case, I would always get "GATT ERROR 0x85" in the nRF connect logs, and the only way to connect to the BlueNRG again was to reset it (it clears its security database, removing all bond information) and to remove bond information from the cellphone. After some time 
[I posted a question on my ST forum](https://community.st.com/s/question/0D50X00009ZE6BiSAL/bluenrg-1-security-example-not-working), but not getting the answer I slowly figured out, that  there is a problem with whitelist because when I turned "undirected connectable" without whitelist, everything worked OK. Let me explain. In the file 
`BlueNRG-1_2-DK-3.0.0/Project/BLE_Examples/BLE_Security/src/peripheral/BLE_Security_Peripheral.c` there is `Make_Connection` function which at first turn general discoverable :

``` cpp
ret = aci_gap_set_discoverable (ADV_IND, 0x100, 0x200, PUBLIC_ADDR, filter, sizeof (local_name), local_name, 0, NULL, 0, 0);
```

with filter ==NO_WHITE_LIST_USE and after bonding it turns undirected connectable like this :

``` cpp
ret = aci_gap_set_undirected_connectable(0x100,0x200,PUBLIC_ADDR, filter);
```

with `filter == WHITE_LIST_FOR_ALL`. The way I see this problem is, that modern 
[cellphones use random private resolvable addresses](http://www.summitdata.com/blog/overview-addressing-privacy-lairds-ble-modules/) which can change anytime, so it may happen that the address BlueNRG has on its whitelist is out of date. The solution is to turn on the controller privacy (introduced in so-called link-layer privacy 1.2 in BLE 4.2) which would resolve addresses in the link-layer tier thus allowing whitelisting. There are more reliable sources on the topic, but the way I understand it is that [I may be wrong here] on the 3rd stage of bonding peripheral and central exchange IRKs between each other (along with other information) and thus the peer device identity is stored in the BlueNRG. This peer device identity consists of peer's device identity address (the "real" address which can be public or static random), the local IRK and peer's IRK. This information is then passed to the controller letting it to resolve random private resolvable addresses from the peer (cellphone) or to "randomize / hide" its own address. [/I may be wrong here]

To turn the controller privacy on I modified 
`aci_gap_init` as so:

``` cpp
ret = aci_gap_init (GAP_PERIPHERAL_ROLE, 0x02, sizeof (deviceName), &service_handle, &dev_name_char_handle, &appearance_char_handle);
```

but it retured error BLE_STATUS_INVALID_PARAMS. The same goes if I wanted to turn "LE secure connections" instead of "legacy pairing" where I got BLE_ERROR_UNSUPPORTED_FEATURE.It turned out that I needed to modify `stack_user_cfg.h` and turn appropriate options on :

``` cpp
#define CONTROLLER_PRIVACY_ENABLED (1U)
#define SECURE_CONNECTIONS_ENABLED (1U)
#define CONTROLLER_MASTER_ENABLED (0U)
#define CONTROLLER_DATA_LENGTH_EXTENSION_ENABLED (0U)
```

In addition I added `stack_user_cfg.c` and `libcrypto.a` to the source base, and was able to go past the aci_gap_init call. Next I wanted to turn on the 
["LE secure connections" which where introduced in BLE 4.2](http://blog.bluetooth.com/bluetooth-pairing-part-4). This is some fancy option which modifies the way the bonding process goes and introduces even more security, yay. I did:

``` cpp
ret = aci_gap_set_authentication_requirement (BONDING, MITM_PROTECTION_NOT_REQUIRED, SC_IS_SUPPORTED, KEYPRESS_IS_NOT_SUPPORTED, 7, 16, USE_FIXED_PIN_FOR_PAIRING, 123456, STATIC_RANDOM_ADDR);`
```

and got `BLE_STATUS_OUT_OF_MEMORY` (0x48) when adding first (and only) custom GATT service. Turns out, that when SC is supported, there is another characteristic added to the Genaral Access service called 
[Central Address Resolution characteristic](https://www.bluetooth.com/specifications/gatt/viewer?attributeXmlFile=org.bluetooth.characteristic.gap.central_address_resolution.xml). So I needed to fire the `BLUENRG1_Wizard.exe` and generate new header file with "privacy - controller" option turned on. This way ATT attributes number was increased by 1 and the out of memory error went away.

![BlueNRG radio parameters wizard](/blog/assets/bluenrg-wizard-privacy-300x253.png)

Next problem was that `aci_gap_set_discoverable` started to return 
`BLE_STATUS_INVALID_PARAMS` (0x42). It turns out, that when host or controller privacy is turned on, 
`Own_Address_Type` parameter can be only 
`RESOLVABLE_PRIVATE_ADDR` or 
`NON_RESOLVABLE_PRIVATE_ADDR`, and conversely when privacy is off, 
`Own_Address_Type` can be only 
`PUBLIC_ADDR` or 
`STATIC_RANDOM_ADDR`. So this way if you want the BlueNRG to resolve private resolvable addresses from the peer, you are also forced to use them on the NRG as well.

This way I was able to run the firmware without any errors, but my cellphone was unable to detect the device 99% of the time. It would occasionally detect it, but on very rare occasions. The problem was solved after... replacing 16MHz High-speed oscillator crystal with 32MHz one. I really do not know where this bug/feature is documented, because I was able to found it only in the 
`Privacy_1_2_Slave_WhiteList.py` script of 
[the BlueNRG GUI application](https://www.st.com/en/embedded-software/stsw-bnrgui.html). After replacing the crystal, I made a change so 
`HS_SPEED_XTAL` macro would preprocess to 1 and 
`SYSCLK_FREQ` to 32000000.

And the last problem I had, which took me almost 2 days to fix was caused by (I think) improper capabilities setup:

``` cpp
ret = aci_gap_set_io_capability (IO_CAP_NO_INPUT_NO_OUTPUT);
ret = aci_gap_set_authentication_requirement (BONDING, MITM_PROTECTION_NOT_REQUIRED, SC_IS_SUPPORTED, KEYPRESS_IS_NOT_SUPPORTED, 7, 16, USE_FIXED_PIN_FOR_PAIRING, 123456, 
STATIC_RANDOM_ADDR);
```

I wanted to bond using the "Just-Works" scheme, but this particular set of calls above made pairing process behave very oddly. I was able to scan for my device (still using nRF connect), and to bond (… -> bond), but it would not appear on the "bonded" list as it usually happened:

![Nrf connect and bonding peculiarity](/blog/assets/nrf-connect-bonding-peculiar-300x267.jpg)

And although I could connect to such oddly bonded device and even reconnect to id multiple times, what I could not achieve was to reconnect after devices disconnected by themselves due to signal loss (i.e. when a cellphone was carried away). In such case, when I brought my cellphone back and turned scanning on, my BlueNRG device would reappear in the "Scanned" list but with different address! Trying to connect to it would return GAT ERROR 0x3d in the nRF logs which means `BLE_ERROR_CONNECTION_END_WITH_MIC_FAILURE` (MIC is some little chunk of bytes added to the payload when privacy is turned on if I remember correctly). To fix this I had to do two things:

* replace `IO_CAP_NO_INPUT_NO_OUTPUT` with `IO_CAP_DISPLAY_ONLY`.	
* reset the phone and find out, that "Bonding" list in the nRF connect was all of a sudden populated with ten or so instances of my BlueNRG device made that day during some of the tests.
And then during pairing, I had to enter a PIN (123456), and after that I was able to do whatever I wanted, disconnect, go out of range, reconnect turn Bluetooth off and on, reset the phone and so on. All worked pretty fine.

Remarks / TODOs:

* My peripheral's (BlueNRG) name isn't shown int the "Scanned" list in nRF connect, and I don't know how to fix this. 	
* I don't know how to enable "Just Works" scheme. 	
* Linux (Ubuntu 18.04) uses static random addresses, so the whole address resolving and privacy thing is not a problem.
  