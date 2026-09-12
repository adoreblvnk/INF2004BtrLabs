# LAB 2: GPIO & Digital Communication

general-purpose input/output (GPIO) configuration, hardware pull resistors, UART serial communication, single-board loopback, deliberate FIFO loss fault analysis & binary packet framing.

---

## hardware overview & equipment

Raspberry Pi Pico W & Maker Pi hardware peripherals.

- Raspberry Pi Pico W: RP2040 microcontroller, dual-core ARM Cortex-M0+, 264 KB SRAM, 26 multifunction GPIO pins (3.3V logic).
- Maker Pi RP2040 / external breadboard components: push buttons on GP20 & GP22, external LEDs on GP15 & GP2-GP5, 330 Ω current-limiting resistors.
- Micro-USB cable & 1x male-to-male jumper wire.

---

## preprocessor defines & macros

preprocessor directives perform text substitution before compilation. Macros created via `#define` eliminate magic numbers & generate inline code without function call overhead. Always wrap macro parameters in parentheses to avoid operator precedence pitfalls.

```c
#define BTN_PIN 20
#define MIN(a, b) (((a) < (b)) ? (a) : (b))
```

---

## task 1: GPIO input (`picow_blink_button.c`)

configure GP20 as an input with internal pull-up resistor to gate LED blinking on GP15.

wiring & logic:
- GP20 button: configured as active-low input. When unpressed, the internal pull-up resistor holds GP20 at logic high (3.3V) & the LED blinks at 250 ms intervals.
- pressing GP20 pulls the line directly to GND (0V, logic low), pausing LED blinking. Releasing the button returns GP20 to high & resumes blinking.
- if using active-high button circuitry, configure the pin in pull-down mode instead (`gpio_set_pulls(BTN_PIN, false, true)`).

![Screenshot of Pull-up NOT Pressed](img/pullup_notpress2.png)
unpressed state: internal pull-up resistor holds GP20 at 3.3V (logic 1).

![Screenshot of Pull-up Pressed](img/pullup_press2.png)
pressed state: button bridges GP20 to GND, pulling input to 0V (logic 0).

note on `gpio_init(BTN_PIN)`:
- `gpio_init()` explicitly assigns the GPIO pad to the RP2040 SIO peripheral. Omitting `gpio_init()` may appear to work because RP2040 reset defaults leave the input buffer enabled, but relying on reset defaults causes silent failures across hardware revisions. Initialise every pin explicitly.

build & flash (macOS):
```sh
# copy SDK import helper
cp ~/pico/pico-sdk/external/pico_sdk_import.cmake .
# copy CMakeLists from bughunt & patch target to picow_blink_button
cp bughunt/CMakeLists.txt CMakeLists.txt
sed -i '' 's/bughunt2_pico\.c/picow_blink_button.c/g; s/bughunt2/picow_blink_button/g; /frame\.c/d' CMakeLists.txt
# configure & compile
mkdir -p build && cd build
cmake -DPICO_BOARD=pico_w ..
make -j8 picow_blink_button
# flash to Pico W (bootsel mode)
cp picow_blink_button.uf2 /Volumes/RPI-RP2
```

windows (powershell):
```powershell
# copy SDK import helper & CMakeLists from bughunt
Copy-Item C:\pico\pico-sdk\external\pico_sdk_import.cmake .
Copy-Item bughunt\CMakeLists.txt CMakeLists.txt
(Get-Content CMakeLists.txt) -replace 'bughunt2', 'picow_blink_button' -replace 'bughunt2_pico.c', 'picow_blink_button.c' -replace '\s*frame.c', '' | Set-Content CMakeLists.txt
# build
New-Item -ItemType Directory -Force -Path build
cd build
cmake -DPICO_SDK_PATH="C:\pico\pico-sdk" ..
cmake --build . --target picow_blink_button
# flash to Pico W
Copy-Item picow_blink_button.uf2 -Destination D:\
```

---

## task 2: GPIO output (`pulse.c`)

generate an asymmetric pulse train on GP15 (1 s high, 2 s low repeating forever).

wiring:
- GP15 -> 330 Ω resistor -> LED anode; LED cathode -> GND.

![Screenshot of GPIO Output](img/gpio_output.png)
GP15 driving external LED with 1 s high / 2 s low periodic waveform.

RP2040 SIO vs CYW43 on-board LED:
- driving GP15 writes directly to the RP2040 SIO register, producing clean nanosecond-scale edge transitions verifiable on an oscilloscope.
- toggling the on-board LED on Pico W (`cyw43_arch_gpio_put`) requires an SPI transaction to the CYW43439 wireless chip, taking several microseconds with nondeterministic timing.

build & flash (macOS):
```sh
# copy CMakeLists from bughunt & patch target to pulse
cp bughunt/CMakeLists.txt CMakeLists.txt
sed -i '' 's/bughunt2_pico\.c/pulse.c/g; s/bughunt2/pulse/g; /frame\.c/d' CMakeLists.txt
# build & flash
mkdir -p build && cd build
cmake -DPICO_BOARD=pico_w ..
make -j8 pulse
cp pulse.uf2 /Volumes/RPI-RP2
```

windows (powershell):
```powershell
# copy CMakeLists from bughunt & patch target to pulse
Copy-Item bughunt\CMakeLists.txt CMakeLists.txt
(Get-Content CMakeLists.txt) -replace 'bughunt2', 'pulse' -replace 'bughunt2_pico.c', 'pulse.c' -replace '\s*frame.c', '' | Set-Content CMakeLists.txt
# build & flash
New-Item -ItemType Directory -Force -Path build
cd build
cmake -DPICO_SDK_PATH="C:\pico\pico-sdk" ..
cmake --build . --target pulse
Copy-Item pulse.uf2 -Destination D:\
```

---

## task 3: UART communication & loopback (pair)

paired UART0 transmission, single-board UART1 loopback & intermittent FIFO loss analysis.

### part A: paired UART0 communication (pair)

cross-board communication between 2 Pico W boards over UART0 at 115200 baud (8 data bits, 1 stop bit, no parity). Pico A runs `hello_uart.c` transmitting characters 'A' through 'Z', while Pico B runs `uart_rx.c` receiving data & printing over USB CDC serial.

extra hardware:
- Pico
- 3x jumper wires

![Screenshot of Connecting 2 Pico W Together](img/p2puart.png)
paired UART wiring setup with crossed TX/RX lines & common ground.

hardware wiring:

| Pico A (Transmitter) | Pico B (Receiver) | Signal | Purpose |
|---|---|---|---|
| Pin 1 (GP0 - TX0) | Pin 2 (GP1 - RX0) | TX $\to$ RX | Pico A transmits to Pico B |
| Pin 2 (GP1 - RX0) | Pin 1 (GP0 - TX0) | RX $\leftarrow$ TX | Bidirectional return link (optional) |
| Pin 3 (GND) | Pin 3 (GND) | Common GND | Mandatory common voltage reference |

partner A execution (Transmitter):

macOS:
```sh
# copy hello_uart.c from pico examples
cp ~/pico/pico-examples/uart/hello_uart/hello_uart.c .
# copy CMakeLists from bughunt & patch target to hello_uart
cp bughunt/CMakeLists.txt CMakeLists.txt
sed -i '' 's/bughunt2_pico\.c/hello_uart.c/g; s/bughunt2/hello_uart/g; /frame\.c/d' CMakeLists.txt
# build & flash to Pico A (bootsel mode)
mkdir -p build && cd build
cmake -DPICO_BOARD=pico_w ..
make -j8 hello_uart
cp hello_uart.uf2 /Volumes/RPI-RP2
```

windows (powershell):
```powershell
# copy hello_uart.c from pico examples
Copy-Item C:\pico\pico-examples\uart\hello_uart\hello_uart.c .
# copy CMakeLists from bughunt & patch target to hello_uart
Copy-Item bughunt\CMakeLists.txt CMakeLists.txt
(Get-Content CMakeLists.txt) -replace 'bughunt2', 'hello_uart' -replace 'bughunt2_pico.c', 'hello_uart.c' -replace '\s*frame.c', '' | Set-Content CMakeLists.txt
# build & flash to Pico A (bootsel mode)
New-Item -ItemType Directory -Force -Path build
cd build
cmake -DPICO_SDK_PATH="C:\pico\pico-sdk" ..
cmake --build . --target hello_uart
Copy-Item hello_uart.uf2 -Destination D:\
```

partner B execution (Receiver):

macOS:
```sh
# copy CMakeLists from bughunt & patch target to uart_rx
cp bughunt/CMakeLists.txt CMakeLists.txt
sed -i '' 's/bughunt2_pico\.c/uart_rx.c/g; s/bughunt2/uart_rx/g; /frame\.c/d' CMakeLists.txt
# build & flash to Pico B (bootsel mode)
mkdir -p build && cd build
cmake -DPICO_BOARD=pico_w ..
make -j8 uart_rx
cp uart_rx.uf2 /Volumes/RPI-RP2
# open serial monitor (Ctrl-A Ctrl-\ to exit)
screen /dev/cu.usbmodem* 115200
```

windows (powershell):
```powershell
# copy CMakeLists from bughunt & patch target to uart_rx
Copy-Item bughunt\CMakeLists.txt CMakeLists.txt
(Get-Content CMakeLists.txt) -replace 'bughunt2', 'uart_rx' -replace 'bughunt2_pico.c', 'uart_rx.c' -replace '\s*frame.c', '' | Set-Content CMakeLists.txt
# build & flash to Pico B (bootsel mode)
New-Item -ItemType Directory -Force -Path build
cd build
cmake -DPICO_SDK_PATH="C:\pico\pico-sdk" ..
cmake --build . --target uart_rx
Copy-Item uart_rx.uf2 -Destination D:\
```

### part B: single-board UART1 loopback

single-board UART1 loopback on GP8 (TX1) & GP9 (RX1) with GP22 button control.

extra hardware:
- 1x jumper wire

![Screenshot of UART Loopback](img/ex2v2.png)
jumper wire linking GP8 directly to GP9 for local loopback verification.

loopback behavior:
- button GP22 unpressed (high): transmits character `'1'` through UART1 every 1 second. The receiver reads `'1'` & prints `'2'` to the USB terminal.
- button GP22 pressed (low): transmits uppercase letters `'A'` through `'Z'` sequentially with 1-second delay (wrapping back to `'A'`). The receiver converts received uppercase characters to lowercase (`'a'`-`'z'`) & prints to USB terminal.

### part C: deliberate intermittent fault FIFO loss analysis

investigate silent UART data loss under high transmission throughput.

reproducing the fault on demand:
- at 1 char/second, transmission succeeds with 0 dropped bytes.
- reduce transmit delay in `hello_uart.c` from 1000 ms down to 10 ms, 1 ms, and finally `sleep_ms(0)` (tight loop).
- observe receiver output: characters are dropped silently without hardware errors or crashes.

hardware failure mechanism:
- the RP2040 UART peripheral contains a 32-entry hardware FIFO (PL011 architecture).
- at 115200 baud, 1 frame (1 start bit, 8 data bits, 1 stop bit = 10 bits) takes $10 / 115200 \approx 86.8\ \mu\text{s}$. A full 32-byte FIFO fills completely in $32 \times 86.8\ \mu\text{s} \approx 2.78\ \text{ms}$.
- when the receiver blocks during slow USB CDC `printf()` output (which takes milliseconds over USB), the UART RX FIFO overflows & incoming bytes are dropped silently.

5 mitigation strategies:
1. UART RX interrupt (IRQ): trigger an interrupt on RX FIFO threshold to drain bytes immediately into a circular software ring buffer, eliminating main-loop blocking latency.
2. hardware flow control (RTS/CTS): enable CTS/RTS handshaking lines (`uart_set_hw_flow(UART_ID, true, true)`) so the transmitter pauses when the receiver FIFO is near full.
3. DMA background transfer: configure an RP2040 DMA channel triggered by `DREQ_UART_RX` to transfer bytes directly from the UART FIFO into SRAM without CPU intervention.
4. dual-core partitioning: assign Core 1 exclusively to drain the UART FIFO into memory while Core 0 handles USB `printf()` display.
5. overrun error flag monitoring: poll the UART Receive Status Register (`uart_get_hw(UART_ID)->rsr & UART_UARTRSR_OE_BITS`) to detect & log data loss rather than failing silently.

---

## task 4: Bug Hunt #2

```sh
cd bughunt
gcc -Wall -Wextra -o bughunt2_host bughunt2_host.c frame.c && ./bughunt2_host
gcc -Wall -Wextra -fsanitize=address,undefined -g -o bughunt2_check bughunt2_host.c frame.c && ./bughunt2_check
```

refer to bughunt readme for more.

---

## challenge: in-place command parser (`cmdparse`)

zero-allocation ASCII command parser (`PING`, `GET TEMP`, `SET LED <index> <state>`) without `malloc` or `strtok()`. Mutates input buffers in-place by writing null terminators over whitespace separators.

```sh
cd cmdparse
gcc -Wall -Wextra -o cmdparse_host cmdparse_host.c cmdparse.c && ./cmdparse_host
```

refer to cmdparse readme for more.

---

## challenge: bitwise 4-bit LED register

represent a 4-bit register across 4 external LEDs on pins GP2, GP3, GP4 & GP5 (initial state `0001` with GP2 active).

- button A: circular shift left (`<<`) with bitwise wrap-around so each press shifts the active LED across GP2 $\to$ GP3 $\to$ GP4 $\to$ GP5 $\to$ GP2.
- button B: toggle the rightmost bit using bitwise XOR (`^ 0x01`) without modifying the remaining 3 LED states.
- use bitwise operators (`|`, `&`, `^`, `~`, `<<`, `>>`) exclusively to update and display the LED state mask.
