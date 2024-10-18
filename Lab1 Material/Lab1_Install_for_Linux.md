System used: Archlinux

## Pre-requirements

- c language server (for completion, diagnostics), e.g. `clangd`
- A code editor that use language server, e.g. `vscode`, `vim/neovim`

## Install Related Softwares

### STM32CubeMX

STM32CubeMX is mainly responsible for generating the project with your configuration.

For Distro like Ubuntu/Debain, you can go to the [ST official site](https://www.st.com.cn/content/st_com/zh/stm32cubemx.html)
Or you can install the software through distro repository

```bash
yay -S stm32cubemx
```

For Arch, you need to modify the AUR repository (I mean, maybe the maintainer doesn't do a good job).
The URL for the repository:https://aur.archlinux.org/packages/stm32cubemx

First clone the repository

```bash
git clone https://aur.archlinux.org/stm32cubemx.git
```

Modify the required jdk version in file `stm32cubemx.sh`
from `exec archlinux-java-run --min 17 -- -jar /opt/stm32cubemx/STM32CubeMX "$@"` to `exec archlinux-java-run --min 17 --max 20 -- -jar /opt/stm32cubemx/STM32CubeMX "$@"`

Then build and install the STM32CubeMX

```bash
makepkg --noconfirm --skipinteg -si
```

Since STM32CubeMX is not compatible with jdk22 (which is the default jdk that arch is currently using), you need to install jdk17 through `yay -S jdk17-openjdk`

Then you can start STM32CubeMX by running `stm32cubemx`, and hopefully, everything is fine.

### Compiler

Use `arm-none-eabi-gcc`

```bash
yay -S arm-none-eabi-gcc
yay -S arm-none-eabi-newlib
```

### Debugger

Use `OpenOCD` to burn and debug STM32 through STLink v2 (the blue USB device provided by us).

```bash
yay -S openocd
```

## Setup Your STM32 Project

Open your STM32CubeMX, follow the instruction of `Lab1.pdf` to configure your project.

**NOTE**: In `Project Manage -> Project -> Project Settings -> Toolchain / IDE`, use `Makefile/CMake`.

Generate the code and go to the project directory (with `Makefile`/`CMakeLists.txt` in the directory).

Then you need to generate the `compile_commands.json` for `clangd` to recognize the project.

### Makefile

```bash
bear -- make
```

### CMake

```bash
cmake -S ./ -B ./build
```

## Build Project

### Makefile

```bash
make
```

Then target binary file is `./build/<Project Name>.bin`

### CMake

```bash
cmake --build ./build
```

Then target binary file is `./build/<Project Name>.elf`

## Load to STM32F103C8T6

Use `OpenOCD` to load the binary file to the board.

```bash
sudo openocd -f /usr/share/openocd/scripts/interface/stlink.cfg -f /usr/share/openocd/scripts/target/stm32f1x.cfg -c "program ./build/<Project Name>.bin reset exit 0x8000000"
```

### Result

```bash
Open On-Chip Debugger 0.12.0
Licensed under GNU GPL v2
For bug reports, read
    http://openocd.org/doc/doxygen/bugs.html
Info : auto-selecting first available session transport "hla_swd". To override use 'transport select <transport>'.
Info : The selected transport took over low-level target control. The results might differ compared to plain JTAG/SWD
Info : clock speed 1000 kHz
Info : STLINK V2J37S7 (API v2) VID:PID 0483:3748
Info : Target voltage: 3.222587
Info : [stm32f1x.cpu] Cortex-M3 r1p1 processor detected
Info : [stm32f1x.cpu] target has 6 breakpoints, 4 watchpoints
Info : starting gdb server for stm32f1x.cpu on 3333
Info : Listening on port 3333 for gdb connections
[stm32f1x.cpu] halted due to debug-request, current mode: Thread
xPSR: 0x01000000 pc: 0x08000dc8 msp: 0x20005000
** Programming Started **
Info : device id = 0x20036410
Info : flash size = 64 KiB
** Programming Finished **
** Resetting Target **
shutdown command invoked
```

**NOTE**: In different Distro, the `cfg` file for `OpenOCD` may locate in different directories. You need to find it by yourselves.

### By the way, if you use CMake

**Note:** When uploading binary file to STM32, it's recommended to use `.bin` file instead of `.elf` file.
Please use the following script to convert the `.elf` to `.bin` and upload.

```bash
cmake -DCMAKE_EXPORT_COMPILE_COMMANDS=ON .
cmake --build  ./build
arm-none-eabi-objcopy  -O binary -S ./build/*.elf ./build/target.bin
sudo openocd -f /usr/share/openocd/scripts/interface/stlink.cfg -f /usr/share/openocd/scripts/target/stm32f1x.cfg -c "program ./build/target.bin reset exit 0x8000000"
```

## Debug

You have three possible choices. I recommend using Ozone.

### openocd+gdb+gdbfrontend(not recommended)

reference:
https://rohanrhu.github.io/gdb-frontend/tutorials/embedded-debugging/

### openocd+vscode+PlatformIO (not recommended)

reference:
https://blog.csdn.net/qq_41757528/article/details/127741620

### Segger Ozone!!!

reference:
https://blog.csdn.net/weixin_41572450/article/details/124710818

Maybe the best debug tool for stm32

To use segger ozone, you need a different linker called jlink (originally we use st-link v2). You need to buy this linker first (maybe on Taobao or Amazon).

Install Ozone through:

```bash
yay -S ozone
```

#### Setup of ozone project:

- Start Ozone
- Choose Device
  - Device: STM32F103C8
  - Register Set: Cortex-M3
  - Peripherials (optional): /opt/SEGGER/Ozone/Config/Peripherals/STM32F103xx.svd
- Connection Settings
  - Target Interface: SWD
  - Target Interface Speed: 4MHz
  - Host Interface: USB
- Program File: select the binary file you have built (`.elf` is recommended).

#### Debug:

set some breakpoints and watch some variables of your interest.
Press the green "power" icon on the upper left corner to start (upload the program and start the debugging process)
Press the blue "play" icon besides "power" to continue.
