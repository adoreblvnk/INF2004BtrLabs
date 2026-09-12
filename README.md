# INF2004 BtrLabs

(unofficial) BtrLabs is a non-AI version of https://github.com/sirfonzie/INF2004 which is easy to read.

features:
- we keep it SHORT!!! yh no fluff, no dramatic metaphors like "the bug wearing a hat"
- we show & explain commands
- separation between OSes, we do NOT mix Linux / MacOS / Windows terminology in our instructions

consolidated lab workspace for INF2004 using the Raspberry Pi Pico W (RP2040) & Pico C/C++ SDK.

---

## quick environment setup

hardware: Maker Pi Pico Base + Pico WH

configure build tools & persistent environment variables.

macOS:
```sh
# install build tools & toolchain
brew install cmake libusb
brew install --cask gcc-arm-embedded
# clone SDK & examples into ~/pico
mkdir -p ~/pico
git clone https://github.com/raspberrypi/pico-sdk.git ~/pico/pico-sdk
git -C ~/pico/pico-sdk submodule update --init
git clone https://github.com/raspberrypi/pico-examples.git ~/pico/pico-examples
# persist environment variables
echo 'export PICO_SDK_PATH=$HOME/pico/pico-sdk' >> ~/.zshrc
echo 'export PICO_BOARD=pico_w' >> ~/.zshrc
source ~/.zshrc
```

windows (powershell):
```powershell
# clone SDK & examples into C:\pico
New-Item -ItemType Directory -Force -Path C:\pico
git clone https://github.com/raspberrypi/pico-sdk.git C:\pico\pico-sdk
git -C C:\pico\pico-sdk submodule update --init
git clone https://github.com/raspberrypi/pico-examples.git C:\pico\pico-examples
# persist user environment variables
[Environment]::SetEnvironmentVariable("PICO_SDK_PATH", "C:\pico\pico-sdk", "User")
[Environment]::SetEnvironmentVariable("PICO_BOARD", "pico_w", "User")
```

---

## lab index

- [LAB 1: Microcontroller and Development Environment](./LAB1/README.md): toolchain setup, operator predictions (`basic.c`), GP15 blink sequence (`blinky.c`), and [Bug Hunt #1](./LAB1/bughunt/README.md).
- [LAB 2: GPIO and Digital Communication](./LAB2/README.md): button input with internal pull-up (`picow_blink_button.c`), pulse train output (`pulse.c`), UART peer-to-peer communication (pair), UART1 loopback & FIFO overflow analysis, in-place command parser challenge (`cmdparse/`), and [Bug Hunt #2](./LAB2/bughunt/README.md).
- [LAB 3: Interrupts & Timers](./LAB3/README.md): hardware GPIO interrupts & callbacks, HC020K IR optical wheel encoder, hardware timer alarms, HC-SR04P ultrasonic distance measurement, debounced GP21 stopwatch, and [Bug Hunt #3](./LAB3/bughunt/README.md).
- [LAB 4: PWM & ADCs](./LAB4/README.md): 8-slice PWM engine & L298N DC motor speed control, 12-bit ADC FIFO sampling on GP26-GP29 & internal temperature sensor, PWM-to-ADC loopback Nyquist/aliasing analysis, and [Bug Hunt #4](./LAB4/bughunt/README.md).
- [LAB 5: FreeRTOS & Real-Time Concepts](./LAB5/README.md): FreeRTOS task scheduling & states, drift elimination with `xTaskDelayUntil()`, single-producer/single-consumer Message Buffers vs multi-producer Queues, mutual exclusion with mutexes & priority inheritance, and [Bug Hunt #5](./LAB5/bughunt/README.md).
- [LAB 6: Optimisation & Debugging](./LAB6/README.md): execution time benchmarking via `pico_time.h` & volatile sinks, power reduction at 48 MHz, PicoProbe / OpenOCD hardware SWD debugging (pair / 2 boards), PID controller debugging (`pid.c`), 4 hand-optimisation pairs ([`optimise/`](./LAB6/optimise/README.md)), and [Bug Hunt #6](./LAB6/bughunt/README.md).
- [GUIDE: Device Driver Development](./GUIDE_DeviceDriver/README.md): reference guide for clean embedded device driver architecture.
