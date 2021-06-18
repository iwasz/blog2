---
layout: post
title: Out Of Tree Zephyr RTOS module
permalink: /electronics/2021-06-18-out-of-tree-zephyr-module.md/
categories: 
- electronics
---
Sometimes it's inconvenient to develop a driver for some peripheral or sensor *in-tree*. As far as I was able to research (bare with me, I'm relatively new to Zephyr), the Zephyr RTOS modules are the way to go in such situations. First, a bunch of links that pointed me in the right direction:

* [A thread](https://lists.zephyrproject.org/g/devel/topic/how_to_build_external_drivers/33154657?p=,,,20,0,0,0::recentpostdate%2Fsticky,,,20,2,0,33154657) stating that "it should not pose a problem, Nordic is doing just that"
* The [nRF Connect SDK: sdk-nrf](https://github.com/nrfconnect/sdk-nrf) the above link is referring to.
* [Stack overflow thread](https://stackoverflow.com/questions/62358554/zephyros-how-to-add-driver-module-to-out-of-tree-project) where a guy replied to himself. Even though this link covers the topic pretty much, I still stumbled upon some problems. See below. 
* [Another out of tree approach](https://github.com/zephyrproject-rtos/zephyr/issues/27181) but overall, this thread is inconclusive.
* ZEPHYR_EXTRA_MODULES [is mentioned here](https://community.platformio.org/t/zephyr-out-of-tree-driver-doesnt-get-compiled/14412).
* [My module](https://github.com/iwasz/example-zephyr-module) this post is about.
* [Acompanying test application](https://github.com/iwasz/example-zephyr-module-app) to try things out.

My [test module](https://github.com/iwasz/example-zephyr-module) contains a driver for LSM6DS3 accelermoeter and gyroscope. This is an older version of LSM6DSL that is already supported by Zephyr, which makes my implementation as easy as changing one macro.

The current version of Zephyr as of today is 2.6.

# The module
Explains what has been copied and what modified. Create the directory structure:

```
+drivers
  +sensor

+dts
  +bindings
    +sensor

+zephyr
CMakeLists.txt
Kconfig
README.md
```

Copy `zephyr/drivers/sensor/lsm6dsl` from the main Zephyr repo to  `drivers/sensor/lsm6ds3` in our directory. 

## Names

Change all the function, macro and file names from **lsm6dsl** to **lsm6ds3**. Remember about `Kconfig` and `CMakeLists.txt`

Change the macro `LSM6DSL_VAL_WHO_AM_I` from 0x6A to 0x69. This is the only "true" modification in the code.

## Build system

Update / provide build system information. A module has to have a Kconfig and CMakeLists.txt in its root directory, so provide one. In my module, these files are as simple as:

`Kconfig`:
```Kconfig
menu "My super zephyr module"

rsource "drivers/Kconfig"

endmenu
```

`CMakeLists.txt`:

```cmake
add_subdirectory(drivers)
```

In the above files I point to the `drivers` directory, so there have to be Kconfig and CMakeLists.txt there as well:

`drivers` directory contains:

CMakeLists.txt : `add_subdirectory_ifdef(CONFIG_SENSOR sensor)`

Kconfig `rsource "sensor/Kconfig"`

---

`sensor` directory contains:

CMakeLists.txt : `add_subdirectory_ifdef(CONFIG_LSM6DS3 lsm6ds3)`

Kconfig : `rsource "lsm6ds3/Kconfig"`

---

You see the pattern.

## Bindings
Copy `st,lsm6dsl-i2c.yaml` and `st,lsm6dsl-spi.yaml` bindings from Zephyr repo to our `dts/bindings/sensor`, and change the file names accordingly (don't forget about the contents). Binding files are found automatically during the build.

## module.yml
Lastly, provide the module description adding the `zephyr/module.yml` :

```yaml
build:
  cmake: .
  kconfig: Kconfig
  settings:
    dts_root: .
```

## Problems I've had
Zephyr build system isn't very verbose (nor precise) when problems occur, so if you ever encounter errors like this:

```cmake
cmake -B build -G Ninja -DBOARD=nucleo_h743zi -DBOARD_ROOT=. -DZEPHYR_EXTRA_MODULES=/home/iwasz/workspace/zephyr-modbus/my-zephyr-module .
Including boilerplate (Zephyr base): /home/iwasz/workspace/zephyr-modbus/zephyrproject/zephyr/cmake/app/boilerplate.cmake
-- Application: /home/iwasz/workspace/zephyr-modbus/zephyr-inclinometer
-- Zephyr version: 2.6.0-rc3 (/home/iwasz/workspace/zephyr-modbus/zephyrproject/zephyr), build: v2.6.0-rc3
-- Found Python3: /usr/bin/python3.9 (found suitable exact version "3.9.4") found components: Interpreter 
-- Found west (found suitable version "0.8.0", minimum required is "0.7.1")
CMake Error at /home/iwasz/workspace/zephyr-modbus/zephyrproject/zephyr/cmake/zephyr_module.cmake:61 (message):
  /home/iwasz/workspace/zephyr-modbus/my-zephyr-module, given in
  ZEPHYR_EXTRA_MODULES, is not a valid zephyr module

Call Stack (most recent call first):
  /home/iwasz/workspace/zephyr-modbus/zephyrproject/zephyr/cmake/app/boilerplate.cmake:183 (include)
  /home/iwasz/workspace/zephyr-modbus/zephyrproject/zephyr/share/zephyr-package/cmake/ZephyrConfig.cmake:24 (include)
  /home/iwasz/workspace/zephyr-modbus/zephyrproject/zephyr/share/zephyr-package/cmake/ZephyrConfig.cmake:35 (include_boilerplate)
  CMakeLists.txt:5 (find_package)


-- Configuring incomplete, errors occurred!
```

It could mean that the `module.yml` is named incorrectly. In my case, I used `yaml` extension instead of `yml`.

--- 

After correcting the problem with `module.yml`, the app started and the analyzer showed that it does something on the I²C, but then it failed in `drivers/sensor/lsm6ds3/lsm6ds3_trigger.c` in the `lsm6ds3_trigger_set` function. The problematic code was:

```c
if (!drv_data->gpio) {
        LOG_ERR ("triggers not supported");
        return -ENOTSUP;
}
```

Clearly, a problem with my overlay, but why? It seems the Zephyr build system did not see my binding files. [As stated here](https://docs.zephyrproject.org/latest/guides/dts/bindings.html?highlight=bindings#where-bindings-are-located) the `module.yml` has to point at the binding files. The resolution is to add:

```yaml
settings:
    dts_root: .
```

# The test application
[Is here](https://github.com/iwasz/example-zephyr-module-app). The application is straightforward. I copied `zephyr/samples/sensor/lsm6dsl`. I added my custom overlay in the `boards` directory to configure what is connected and to which pins:

```dts
&i2c2 {
        pinctrl-0 = <&i2c2_scl_pb10 &i2c2_sda_pb11>;
        status = "okay";
        clock-frequency = <I2C_BITRATE_FAST>;

        lsm6ds3@6b {
                compatible = "st,lsm6ds3";
                reg = <0x6b>;
                irq-gpios = <&gpiod 11 GPIO_ACTIVE_HIGH>;
                label = "LSM6DS3";
        };
};
```

I modified the `prj.conf` to enable my implementation instead of theirs (again I changed from LSM6DSL to LSM6DS3).

Finally I changed some (but not all) names in the C code from LSM6DSL to LSM6DS3:

* `device_get_binding (DT_LABEL (DT_INST (0, st_lsm6ds3)))`
* `CONFIG_LSM6DS3_TRIGGER`

# Build
After these modification I was left with:

* The Zephyr itself (usual directory `zephyrproject` + SDK)
* My application in `my-zephyr-module-app`.
* My module in `my-zephyr-module`

To build this i simply have to :

```sh
cd my-zephyr-module-app
cmake -B build -G Ninja -DBOARD=nucleo_h743zi -DBOARD_ROOT=. -DZEPHYR_EXTRA_MODULES=/home/iwasz/workspace/zephyr-modbus/my-zephyr-module .
ninja -C build/
west flash
```

# TODO
Make vscode happy. The IDE is overwhelmed by the project split into 3 separate directories or I don't know how to configure it. Let me know in the commensts you whereabouts on this. Also I do not know (an I've tried) how to have the module included inside the application directory. 
