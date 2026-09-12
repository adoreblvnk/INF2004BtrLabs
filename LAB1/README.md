# LAB 1: Microcontroller and Development Environment

environment setup & foundational C operations for Raspberry Pi Pico W.

---

## hardware overview

Raspberry Pi Pico & Pico W board layouts.

![Raspberry Pi Pico Pinout](img/pico%20pinout.png)
- Raspberry Pi Pico: RP2040 microcontroller, dual-core ARM Cortex-M0+, 264 KB SRAM, 26 multifunction GPIO pins (3.3V logic).

![Raspberry Pi Pico W Pinout](https://www.raspberrypi.com/documentation/microcontrollers/images/picow-pinout.svg)
- Raspberry Pi Pico W: includes CYW43439 wireless chip. Note that on-board LED is connected directly to the wireless chip via SPI rather than RP2040 SIO GPIO.

---

## environment setup

install build toolchain & configure shell environment variables.

macOS:
```sh
# install build tools, serial monitor & ARM GCC toolchain
brew install cmake libusb
brew install --cask gcc-arm-embedded
# create pico directory & clone Pico SDK with submodules
mkdir -p ~/pico
git clone https://github.com/raspberrypi/pico-sdk.git ~/pico/pico-sdk
git -C ~/pico/pico-sdk submodule update --init
# clone pico examples
git clone https://github.com/raspberrypi/pico-examples.git ~/pico/pico-examples
# export environment variables permanently (prevents terminal amnesia)
echo 'export PICO_SDK_PATH=$HOME/pico/pico-sdk' >> ~/.zshrc
echo 'export PICO_BOARD=pico_w' >> ~/.zshrc
# reload shell configuration
source ~/.zshrc
```

windows (powershell):
```powershell
# clone Pico SDK & submodules
New-Item -ItemType Directory -Force -Path C:\pico
git clone https://github.com/raspberrypi/pico-sdk.git C:\pico\pico-sdk
git -C C:\pico\pico-sdk submodule update --init
# clone pico examples
git clone https://github.com/raspberrypi/pico-examples.git C:\pico\pico-examples
# set permanent user environment variables
[Environment]::SetEnvironmentVariable("PICO_SDK_PATH", "C:\pico\pico-sdk", "User")
[Environment]::SetEnvironmentVariable("PICO_BOARD", "pico_w", "User")
```

---

## build pipeline overview

compilation & flashing workflow.

![Build Overview](img/overview.png)
- C source files are processed by CMake and compiled using ARM GCC (`arm-none-eabi-gcc`) into ELF/UF2 binaries, which are copied onto the Pico mass storage volume in BOOTSEL mode.

---

## task 1: Predict, then run (`basic.c`)

verify evaluation rules for arithmetic, logical, relational, increment/decrement & bitwise operators in C before running code.

key operator predictions:
- `b / a` (20 / 10 = 2) & `b % a` (20 % 10 = 0): integer division truncates fractions toward zero.
- `c++` vs `++c`: `c` begins at 5. `c++` evaluates to 5 (post-increment, `c` becomes 6). `++c` evaluates to 7 (pre-increment, `c` becomes 7). `c--` evaluates to 7 (`c` becomes 6). `--c` evaluates to 5 (`c` becomes 5).
- `~p` where `p = 5`: bitwise NOT on signed 32-bit integer inverts all bits, producing two's complement value `-6`.
- relational expressions (`a > b`, `a == b`): evaluate to integer `0` (false) or `1` (true).

build & run on host:
```sh
# compile directly with host gcc
gcc -Wall -Wextra -o basic basic.c
# execute & compare predictions against terminal output
./basic
```

build & flash to Pico W:
```sh
# copy sdk import helper from ~/pico
cp ~/pico/pico-sdk/external/pico_sdk_import.cmake .
# copy CMakeLists from bughunt & patch target name
cp bughunt/CMakeLists.txt CMakeLists.txt
sed -i '' 's/bughunt1/basic/g' CMakeLists.txt
# build target
mkdir -p build && cd build
cmake -DPICO_BOARD=pico_w ..
make -j8 basic
# flash via bootsel mode
cp basic.uf2 /Volumes/RPI-RP2
# open serial monitor
screen /dev/tty.usbmodem* 115200
```

---

## task 2: Blinky warm-up (`blinky.c`)

blink external LED on GP15 across 11 doubling intervals (1 ms to 1024 ms), resetting back to 1 ms upon reaching 2048 ms.

wiring:
- GP15 -> 330 Ω resistor -> LED anode; LED cathode -> GND
- on Pico W, the on-board LED is connected via SPI to CYW43 WiFi SoC, so driving GP15 provides direct RP2040 GPIO output.

defects to fix (3 total):
1. line 29: `sleep_ms(a<<1)` computes shift without updating variable `a`. Update `a` so that the value doubles on each iteration.
2. line 33: off-delay is doubled instead of matching the on-delay. Ensure both on and off durations use identical delay periods.
3. line 37: `if(a=2048) a==0;` mixes up assignment `=` and comparison `==` operators. Check whether `a` reaches 2048 and reset it back to 1.

build & flash:
```sh
# copy sdk import helper
cp ~/pico/pico-sdk/external/pico_sdk_import.cmake .
# copy CMakeLists from bughunt & patch target to blinky
cp bughunt/CMakeLists.txt CMakeLists.txt
sed -i '' 's/bughunt1/blinky/g' CMakeLists.txt
# configure and build
rm -rf build && mkdir -p build && cd build
cmake -DPICO_BOARD=pico_w ..
make -j8 blinky
# flash to Pico W (bootsel mode)
cp blinky.uf2 /Volumes/RPI-RP2
```

windows (powershell):
```powershell
# copy import helper & CMakeLists
Copy-Item C:\pico\pico-sdk\external\pico_sdk_import.cmake .
Copy-Item bughunt\CMakeLists.txt CMakeLists.txt
(Get-Content CMakeLists.txt) -replace 'bughunt1', 'blinky' | Set-Content CMakeLists.txt
# build
New-Item -ItemType Directory -Force -Path build
cd build
cmake -DPICO_SDK_PATH="C:\pico\pico-sdk" ..
cmake --build . --target blinky
# flash to Pico W (bootsel mode)
Copy-Item blinky.uf2 -Destination D:\
```

verify:
- LED delay sequence doubles steadily: 1 ms, 2 ms, 4 ms, 8 ms, 16 ms, 32 ms, 64 ms, 128 ms, 256 ms, 512 ms, 1024 ms, then resets to 1 ms.

---

## task 3: Bug Hunt #1

refer to bughunt readme for more.
