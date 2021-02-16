# ARM Tutorial

C programming on the Raspberry Pi PICO

Comprehensive getting started information is [available](https://datasheets.raspberrypi.org/pico/getting_started_with_pico.pdf) via the raspberry pi website].

The RPi foundation documentation is based on the development host (where you develop code) being a raspberry pi. In this tutorial we develop on standard desktop. You'll need a JTAG device that's compatible with OpenOCD in order to be able to develop properly on your desktop however. The Raspberry Pi guide above using GPIO as the JTAG connection for debugging.

## Quickstart

Getting started quickly. Here, on Fedora 33.

### Toolchain

The toolchain required by the PICO SDK is `gcc-arm-none-eabi` this is a GCC compiler that supports the Cortex M0+ processor architecture. The compiler is generally available in the defaault repositories of all the major Linux distributions. You can go ahead and install the toolchain:

For Fedora I suggest the following packages are installed:

```
$ dnf install arm-none-eabi-gcc-cs \
        arm-none-eabi-binutils-cs \
        arm-none-eabi-gcc-cs-c++ \
        arm-none-eabi-newlib \
        cmake \
        make \
        doxygen \
        gcc
```

Test the toolchain:

```
$ arm-none-eabi-gcc --version

arm-none-eabi-gcc (Fedora 10.2.0-2.fc33) 10.2.0
Copyright (C) 2020 Free Software Foundation, Inc.
This is free software; see the source for copying conditions.  There is NO
warranty; not even for MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.
```

### Get the SDK

You'll also need the RPI, straight from the RPI getting started... 

```
git clone -b master https://github.com/raspberrypi/pico-sdk.git
cd pico-sdk && export PICO_SDK_PATH=$(pwd)
git submodule update --init
cd ..
git clone -b master https://github.com/raspberrypi/pico-examples.git
```

Make sure you export the PICO_SDK_PATH above so the following cmake command works.

In order to build all of the examples at once, you can just do the following.

```
cd pic-examples && mkdir build && cd build
cmake -DCMAKE_BUILD_TYPE=Debug ../
make -j$(nproc)
```

If you've got a slow computer you may just want to build the led blink project or something. To do that, you could do:

```
cd pic-examples && mkdir build && cd build
cmake -DCMAKE_BUILD_TYPE=Debug ../
cd blink && make -j$(nproc)
```

### Load the Code

The RP2040 has a USB bootloader built in. In order to upload firmware, we plug the pic board into the computer and copy one of the example `.u2f` files that we just built to the mass storage device that pops up.

Keep your finger on the `BOOTSEL` button as you plug the RPi pico into your computer and then you'll see the RPi in the file explorer. If your linux distribution doesn't automount USB then you'll have to mount it first.

From `pico-examples/build/blink` copy `blink.u2f` to the mounted filesystem of the pico.

When the code has been written to flash the bootloader in the pico will automatically disconnect the USB connection and reboot to start running the new code.

You see the blinking LED on the RPi pico. :)

Now, let's see how we can develop

