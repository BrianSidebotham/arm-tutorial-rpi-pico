# Development Getting Started

In order to do some development, we could do with an IDE. The IDE should have basic development features that we expect these days - syntax highlighting, debug with breakpoints and watches, etc.

Visual Studio Code can be used for the IDE to get a good development experience, but we first need to give Visual Studio Code some way of communicating with our external hardware.

The goal here is to be able to build and debug directly on the PICO hardware in Visual Studio Code using a JTAG device.

I'm starting with Fedora 33, the following packages installed:

```shell
$ sudo dnf install ncurses-compat-libs gcc make cmake autoconf automake libtool libftdi-devel libusb-devel texinfo
```

## JTAG

To support Debugging we'll need some sort of debug connection. The industry standard these days is JTAG. We will use a second PICO setup as a JTAG interface for debugging our PICO.

So, you need two PICOs in order to debug properly.

### Program a Picoprobe

Configuring a PICO to run as a JTAG interface is clled a Picoprobe by RPi.

We need to build OpenOCD ourselves in in order to get support for the picoprobe. This can be done fairly easily. You can see the instructions in the [official RPi docuementation](https://datasheets.raspberrypi.org/pico/getting-started-with-pico.pdf), Appendix A.

Fedora 33 example:

```shell
$ git clone https://github.com/raspberrypi/openocd.git --branch picoprobe --depth=1
$ cd openocd
$ ./bootstrap
$ ./configure --enable-picoprobe
$ make -j$(nproc)
$ sudo make install
```

Grab the Picoprobe U2F file which is the JTAG interface software a PICO needs to run in order to provide a JTAG interface for debugging the second PICO:

```shell
$ wget https://www.raspberrypi.org/documentation/pico/getting-started/static/fec949af3d02572823529a1b8c1140a7/picoprobe.uf2
```

Plug a PICO in to your computer whilst holding down the boot switch, and copy `picoprobe.u2f` to the drive that it creates.

The bootloader will write the U2F file to the PICO's Flash and then it will disconnect and reboot. That's how the bootloader for the PICO works. The onboard LED should be lit

### Connect a Picoprobe

Wire the picoprobe and PICO board under test as in the following diagram and photo.

![PICO_SVD](/images/part-1-pico-svd.jpg)

### Test the Picoprobe

You should be able to test connecting to the

On most Linux systems you might experience permissions issues. On Fedora (and most others) you can allow yourself to have permissions to the USB connection of the Picoprobe. Do this rather than trying to run Visual Studio Code or Picoprobe as root!

In `/etc/udev/rules.d/99-picoprobe.cfg`:

```
SUBSYSTEM=="usb", ATTRS{idVendor}=="2e8a", ATTRS{idProduct}=="0004", MODE="0666"
```

Then unplug and re-plug the Picoprobe if you already had it plugged in.

The following should not give you a `LIBUSB_ACCESS_ERROR`. If it does, you must solve that issue first before continuing.

```shell
openocd -c "gdb_port 50000"  -f interface/picoprobe.cfg -f target/rp2040.cfg
Open On-Chip Debugger 0.10.0+dev-geb22ace-dirty (2021-02-14-16:43)
Licensed under GNU GPL v2
For bug reports, read
	http://openocd.org/doc/doxygen/bugs.html
Info : only one transport option; autoselect 'swd'
Warn : Transport "swd" was already selected
adapter speed: 5000 kHz

Info : Hardware thread awareness created
Info : Hardware thread awareness created
Info : RP2040 Flash Bank Command
Info : Listening on port 6666 for tcl connections
Info : Listening on port 4444 for telnet connections
Info : clock speed 5000 kHz
Info : SWD DPIDR 0x0bc12477
Info : SWD DLPIDR 0x00000001
Info : SWD DPIDR 0x0bc12477
Info : SWD DLPIDR 0x10000001
Info : rp2040.core0: hardware has 4 breakpoints, 2 watchpoints
Info : rp2040.core1: hardware has 4 breakpoints, 2 watchpoints
Info : starting gdb server for rp2040.core0 on 50000
Info : Listening on port 50000 for gdb connections
```

## Configure Visual Studio Code

First, we need a toolchain. You can install the one packaged for your linux distribution if you want, but I prefer to grab a toolhcain from ARM and then install it locally just for user account in [`.local`](https://www.freedesktop.org/software/systemd/man/file-hierarchy.html#Home%20Directory):

```
~$ wget https://armkeil.blob.core.windows.net/developer/Files/downloads/gnu-rm/10-2020q4/gcc-arm-none-eabi-10-2020-q4-major-x86_64-linux.tar.bz2
~$ mkdir -p ~/.local && cd ~/.local
~$ tar --strip-components 1 -xf ~/gcc-arm-none-eabi-10-2020-q4-major-x86_64-linux.tar.bz2
```

>**NOTE**: Not all distributions come with an ARM GDB package which is why I like the consistency of grabbing a toolchain from ARM.

Test that your toolchain is working OK (Which ever way you installed it)

```shell
$ arm-none-eabi-gcc --version
$ arm-none-eabi-gdb --version
```

Install Visual Studio Code by any means necessary for your platform and make sure you install the following plugins:

```shell
$ code --install-extension marus25.cortex-debug
$ code --install-extension ms-vscode.cmake-tools
$ code --install-extension ms-vscode.cpptools
```

## Starting Visual Studio Code

In order for the CMAKE system to work, the RPI foundation have used an environment variable in order to point to the SDK. Export it to your environment before starting Visual Code from the same environment. You could also add this to your shell environment by using `~/.bashrc` or similar. It is recommended that you add it to your `~/.bashrc` or if for every user on your computer, your `/etc/profile` instead.

```shell
$ export PICO_SDK_PATH=/home/bjs/pico-sdk
```

Alternatively, you can set the environment variable in the CMake Extension configuration in Visual Studio Code. Make sure you set it in the Configure section so the environment variable is passed when the project is configured.

![PICO_SDK_PATH](/images/part-1-configure-pico-sdk-path.jpg)

Open the `./part-1-development-getting-started` directory in Visual Studio Code.:

```shell
$ export PICO_SDK_PATH=/home/bjs/pico-sdk; cd ./part-1-development-getting-started && code ./
```

>**NOTE:** If you try to set the PICO_SDK_PATH in the CMake extension environment path section you come unstuck because during debugging the extension in use is Cortex-Debug and therefore the PICO_SDK_PATH environment variable will not be available and then you won't be able to load the SVD file which includes the Register IO mappings and is therefore very useful for debugging.

Select the correct toolchain from the status bar at the bottom of Visual Studio Code, and do the same to select the Debug CMake target. This will enable debug symboles in the build process.

![CMAKE_BUILD_TYPE](/images/part-1-cmake-debug-build-type.jpg)

When you select the CMake target, Visual Studio Code will go ahead and configure the project for you. In the output window you should see the CMake output along the lines of:

```
[main] Configuring folder: part-1-development-getting-started
[proc] Executing command: /usr/bin/cmake --no-warn-unused-cli -DCMAKE_EXPORT_COMPILE_COMMANDS:BOOL=TRUE -DCMAKE_BUILD_TYPE:STRING=Debug -DCMAKE_C_COMPILER:FILEPATH=/home/bjs/.local/bin/arm-none-eabi-gcc-10.2.1 -H/home/bjs/arm-tutorial-rpi-pico/part-1-development-getting-started -B/home/bjs/arm-tutorial-rpi-pico/part-1-development-getting-started/build -G "Unix Makefiles"
[cmake] Not searching for unused variables given on the command line.
[cmake] Using PICO_SDK_PATH from environment ('/home/bjs/pico-sdk')
[cmake] Pico SDK is located at /home/bjs/pico-sdk
[cmake] Defaulting PICO_PLATFORM to rp2040 since not specified.
[cmake] Defaulting PICO platform compiler to pico_arm_gcc since not specified.
[cmake] PICO compiler is pico_arm_gcc
[cmake] PICO_GCC_TRIPLE defaulted to arm-none-eabi
[cmake] -- The C compiler identification is GNU 10.2.1
[cmake] -- The CXX compiler identification is GNU 10.2.1
[cmake] -- The ASM compiler identification is GNU
[cmake] -- Found assembler: /home/bjs/.local/bin/arm-none-eabi-gcc
[cmake] Using regular optimized debug build (set PICO_DEOPTIMIZED_DEBUG=1 to de-optimize)
[cmake] Defaulting PICO target board to pico since not specified.
[cmake] Using board configuration from /home/bjs/pico-sdk/src/boards/include/boards/pico.h
[cmake] -- Found Python3: /usr/bin/python3.9 (found version "3.9.1") found components: Interpreter
[cmake] TinyUSB available at /home/bjs/pico-sdk/lib/tinyusb/src/portable/raspberrypi/rp2040; adding USB support.
[cmake] Compiling TinyUSB with CFG_TUSB_DEBUG=1
[cmake] -- Found Doxygen: /usr/bin/doxygen (found version "1.8.20") found components: doxygen dot
[cmake] ELF2UF2 will need to be built
[cmake] -- Configuring done
[cmake] -- Generating done
[cmake] -- Build files have been written to: /home/bjs/arm-tutorial-rpi-pico/part-1-development-getting-started/build
```

Next up, click the CMake icon in the left icon tray in Visual Studio Code. Select the `blink.elf` target and click the build icon

![CMAKE_SELECT_BUILD](/images/part-1-select-blink-cmake-for-build.jpg)

Building the project includes a lot of output even though we've only got visibility of one, very simple c source file. This is because the SDK gets built and linked to the project. The complete make output for CMake is something like:

```
[main] Building folder: part-1-development-getting-started blink
[build] Starting build
[proc] Executing command: /usr/bin/cmake --build /home/bjs/part-1-development-getting-started/build --config Debug --target blink -- -j 26
[build] Scanning dependencies of target ELF2UF2Build
[build] Scanning dependencies of target bs2_default
[build] [  1%] Creating directories for 'ELF2UF2Build'
[build] [  3%] Building ASM object pico_sdk/src/rp2_common/boot_stage2/CMakeFiles/bs2_default.dir/boot2_w25q080.S.obj
[build] [  5%] Linking ASM executable bs2_default.elf
[build] [  6%] No download step for 'ELF2UF2Build'
[build] [  6%] Built target bs2_default
[build] Scanning dependencies of target bs2_default_bin
[build] [  8%] No update step for 'ELF2UF2Build'
[build] [ 10%] Generating bs2_default.bin
[build] [ 10%] Built target bs2_default_bin
[build] [ 11%] No patch step for 'ELF2UF2Build'
[build] Scanning dependencies of target bs2_default_padded_checksummed_asm
[build] [ 13%] Generating bs2_default_padded_checksummed.S
[build] [ 15%] Performing configure step for 'ELF2UF2Build'
[build] [ 15%] Built target bs2_default_padded_checksummed_asm
[build] -- The C compiler identification is GNU 10.2.1
[build] -- The CXX compiler identification is GNU 10.2.1
[build] -- Detecting C compiler ABI info
[build] -- Detecting C compiler ABI info - done
[build] -- Check for working C compiler: /usr/bin/cc - skipped
[build] -- Detecting C compile features
[build] -- Detecting C compile features - done
[build] -- Detecting CXX compiler ABI info
[build] -- Detecting CXX compiler ABI info - done
[build] -- Check for working CXX compiler: /usr/bin/c++ - skipped
[build] -- Detecting CXX compile features
[build] -- Detecting CXX compile features - done
[build] -- Configuring done
[build] -- Generating done
[build] -- Build files have been written to: /home/bjs/part-1-development-getting-started/build/elf2uf2
[build] [ 16%] Performing build step for 'ELF2UF2Build'
[build] Scanning dependencies of target elf2uf2
[build] [ 50%] Building CXX object CMakeFiles/elf2uf2.dir/main.cpp.o
[build] [100%] Linking CXX executable elf2uf2
[build] [100%] Built target elf2uf2
[build] [ 18%] No install step for 'ELF2UF2Build'
[build] [ 20%] Completed 'ELF2UF2Build'
[build] [ 20%] Built target ELF2UF2Build
[build] Scanning dependencies of target blink
[build] [ 22%] Building C object CMakeFiles/blink.dir/home/bjs/pico-sdk/src/rp2_common/hardware_gpio/gpio.c.obj
[build] [ 28%] Building C object CMakeFiles/blink.dir/home/bjs/pico-sdk/src/rp2_common/pico_stdlib/stdlib.c.obj
[build] [ 32%] Building C object CMakeFiles/blink.dir/home/bjs/pico-sdk/src/rp2_common/pico_platform/platform.c.obj
[build] [ 27%] Building C object CMakeFiles/blink.dir/home/bjs/pico-sdk/src/rp2_common/hardware_claim/claim.c.obj
[build] [ 32%] Building C object CMakeFiles/blink.dir/home/bjs/pico-sdk/src/rp2_common/hardware_sync/sync.c.obj
[build] [ 32%] Building C object CMakeFiles/blink.dir/blink.c.obj
[build] [ 33%] Building C object CMakeFiles/blink.dir/home/bjs/pico-sdk/src/common/pico_sync/lock_core.c.obj
[build] [ 37%] Building C object CMakeFiles/blink.dir/home/bjs/pico-sdk/src/common/pico_time/timeout_helper.c.obj
[build] [ 37%] Building ASM object CMakeFiles/blink.dir/home/bjs/pico-sdk/src/rp2_common/hardware_divider/divider.S.obj
[build] [ 40%] Building C object CMakeFiles/blink.dir/home/bjs/pico-sdk/src/common/pico_time/time.c.obj
[build] [ 30%] Building C object CMakeFiles/blink.dir/home/bjs/pico-sdk/src/rp2_common/hardware_timer/timer.c.obj
[build] [ 40%] Building C object CMakeFiles/blink.dir/home/bjs/pico-sdk/src/rp2_common/hardware_uart/uart.c.obj
[build] [ 44%] Building C object CMakeFiles/blink.dir/home/bjs/pico-sdk/src/common/pico_util/datetime.c.obj
[build] [ 44%] Building C object CMakeFiles/blink.dir/home/bjs/pico-sdk/src/common/pico_sync/sem.c.obj
[build] [ 45%] Building C object CMakeFiles/blink.dir/home/bjs/pico-sdk/src/common/pico_sync/mutex.c.obj
[build] [ 49%] Building C object CMakeFiles/blink.dir/home/bjs/pico-sdk/src/rp2_common/hardware_watchdog/watchdog.c.obj
[build] [ 50%] Building C object CMakeFiles/blink.dir/home/bjs/pico-sdk/src/common/pico_sync/critical_section.c.obj
[build] [ 50%] Building C object CMakeFiles/blink.dir/home/bjs/pico-sdk/src/common/pico_util/pheap.c.obj
[build] [ 52%] Building C object CMakeFiles/blink.dir/home/bjs/pico-sdk/src/rp2_common/pico_runtime/runtime.c.obj
[build] [ 54%] Building C object CMakeFiles/blink.dir/home/bjs/pico-sdk/src/common/pico_util/queue.c.obj
[build] [ 55%] Building C object CMakeFiles/blink.dir/home/bjs/pico-sdk/src/rp2_common/hardware_xosc/xosc.c.obj
[build] [ 57%] Building C object CMakeFiles/blink.dir/home/bjs/pico-sdk/src/rp2_common/hardware_clocks/clocks.c.obj
[build] [ 61%] Building ASM object CMakeFiles/blink.dir/home/bjs/pico-sdk/src/rp2_common/hardware_irq/irq_handler_chain.S.obj
[build] [ 61%] Building C object CMakeFiles/blink.dir/home/bjs/pico-sdk/src/rp2_common/hardware_vreg/vreg.c.obj
[build] [ 64%] Building C object CMakeFiles/blink.dir/home/bjs/pico-sdk/src/rp2_common/hardware_pll/pll.c.obj
[build] [ 64%] Building C object CMakeFiles/blink.dir/home/bjs/pico-sdk/src/rp2_common/hardware_irq/irq.c.obj
[build] [ 66%] Building C object CMakeFiles/blink.dir/home/bjs/pico-sdk/src/rp2_common/pico_printf/printf.c.obj
[build] [ 67%] Building ASM object CMakeFiles/blink.dir/home/bjs/pico-sdk/src/rp2_common/pico_bit_ops/bit_ops_aeabi.S.obj
[build] [ 69%] Building C object CMakeFiles/blink.dir/home/bjs/pico-sdk/src/rp2_common/pico_bootrom/bootrom.c.obj
[build] [ 71%] Building C object CMakeFiles/blink.dir/home/bjs/pico-sdk/src/rp2_common/pico_double/double_math.c.obj
[build] [ 72%] Building ASM object CMakeFiles/blink.dir/home/bjs/pico-sdk/src/rp2_common/pico_divider/divider.S.obj
[build] [ 74%] Building C object CMakeFiles/blink.dir/home/bjs/pico-sdk/src/rp2_common/pico_double/double_init_rom.c.obj
[build] [ 76%] Building ASM object CMakeFiles/blink.dir/home/bjs/pico-sdk/src/rp2_common/pico_double/double_v1_rom_shim.S.obj
[build] [ 77%] Building ASM object CMakeFiles/blink.dir/home/bjs/pico-sdk/src/rp2_common/pico_double/double_aeabi.S.obj
[build] [ 79%] Building ASM object CMakeFiles/blink.dir/home/bjs/pico-sdk/src/rp2_common/pico_float/float_aeabi.S.obj
[build] [ 81%] Building ASM object CMakeFiles/blink.dir/home/bjs/pico-sdk/src/rp2_common/pico_int64_ops/pico_int64_ops_aeabi.S.obj
[build] [ 83%] Building C object CMakeFiles/blink.dir/home/bjs/pico-sdk/src/rp2_common/pico_float/float_init_rom.c.obj
[build] [ 84%] Building C object CMakeFiles/blink.dir/home/bjs/pico-sdk/src/rp2_common/pico_float/float_math.c.obj
[build] [ 86%] Building ASM object CMakeFiles/blink.dir/home/bjs/pico-sdk/src/rp2_common/pico_float/float_v1_rom_shim.S.obj
[build] [ 88%] Building C object CMakeFiles/blink.dir/home/bjs/pico-sdk/src/rp2_common/pico_malloc/pico_malloc.c.obj
[build] [ 89%] Building ASM object CMakeFiles/blink.dir/home/bjs/pico-sdk/src/rp2_common/pico_mem_ops/mem_ops_aeabi.S.obj
[build] [ 91%] Building ASM object CMakeFiles/blink.dir/home/bjs/pico-sdk/src/rp2_common/pico_standard_link/crt0.S.obj
[build] [ 93%] Building CXX object CMakeFiles/blink.dir/home/bjs/pico-sdk/src/rp2_common/pico_standard_link/new_delete.cpp.obj
[build] [ 94%] Building C object CMakeFiles/blink.dir/home/bjs/pico-sdk/src/rp2_common/pico_standard_link/binary_info.c.obj
[build] [ 96%] Building C object CMakeFiles/blink.dir/home/bjs/pico-sdk/src/rp2_common/pico_stdio/stdio.c.obj
[build] [ 98%] Building C object CMakeFiles/blink.dir/home/bjs/pico-sdk/src/rp2_common/pico_stdio_uart/stdio_uart.c.obj
[build] [100%] Linking CXX executable blink.elf
[build] [100%] Built target blink
[build] Build finished with exit code 0
```

Loads of stuff to get an LED blinking. But also loads of stuff that allows us to just get on with the job in hand straight away.

## Debugging in Visual Studio Code

Now that we have the project built, we can go ahead and debug the project running on the Raspberry Pi PICO.

Click the Debug icon on the left hand icon palette in Visual Studio Code.

At the top of the tab click the green triangle which is labelled `Pico Debug`. This will start using the settings in `.vscode/launch.json` to start the debug process. This interfaces to the `cortex-debug` extension that we installed earlier.

The debugger automatically stops at the start of the `main()` function.

![VISUAL_DEBUG](/images/part-1-debug-visual-studio-code.jpg)

Click the continue button (blue triangle) in the debug controls at the top of the Visual Studio window which hovers above the source file tabs.

The LED on the second PICO starts to blink. Click pause and the target will be paused and Visual Studio Code will show you which peice of code was running at the time it paused (most likely somewhere in the SDK).

As per most debuggers these days we get a variables window that includes local and global variables (Where they're not optimised out anyway), and the ability to watch variables too. The Call Stack can show us how deep the current call stack is and what we're currently executing.

There is a great window called Cortex Peripherals which includes all of the IO registers so we can easily see how the IO registers are looking. This is something that a lot of open source embedded editors never get round to including. It's extemely useful.

There's a register view too so we can see the current value of the registers (The PICO has two M0 cores, so not sure how well that pans out as it looks like the Corex-Debug doesn't include the full set of registers we may be interested in.)

Breakpoints can be set in the gutter of the source code. As you hover over the source code gutter (by the line numbers) you can click to set a red dot which indicates a breakpoint.

Go ahead and set a breakpoint on Line 16 and then press Continue again in the debug controls.

![VISUAL_BREAKPOINT](/images/part-1-breakpoint-hit-visual-studio-code.jpg)

You can step-over or step into the code to see it operate in more detail and see the registers, etc. update while you do.


