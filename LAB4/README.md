# LAB 4: Pulse Width Modulation & Analog-to-Digital Conversion

hardware configuration, PWM signal generation, ADC sampling, & aliasing analysis on Raspberry Pi Pico W.

---

## hardware & peripheral overview

RP2040 PWM architecture & ADC channels overview.

![PWM GPIO](img/pwmgpio.png)
- RP2040 PWM peripheral: features 8 identical slices (0 to 7) generating up to 16 controllable PWM outputs across all 30 GPIO pins. Each slice drives 2 independent output channels (A & B).

![L298N Pinout](img/l298n_pinout.png)
- L298N dual H-bridge motor driver: accepts high-power motor supplies (up to 35V) & low-voltage TTL logic. Speed is modulated via ENA/ENB enable pins (PWM input with jumpers removed); directional rotation is controlled via IN1-IN4 logic inputs.

---

## task 1: Pulse Width Modulation & L298N DC Motor

configure hardware PWM slices to drive DC motor speed & direction via L298N H-bridge module.

extra hardware:
- L298N motor driver module
- DC motor
- external battery pack / DC power supply
- 5x jumper wires

PWM frequency & duty cycle calculations:
- PWM output frequency is governed by system clock frequency ($f_{\text{sys}} = 125\text{ MHz}$), clock divider (`clkdiv`), & counter wrap value (`wrap`):

$$f_{\text{PWM}} = \frac{f_{\text{sys}}}{\text{clkdiv} \times (\text{wrap} + 1)}$$

- the hardware counter counts from 0 up to `wrap` inclusive, taking $\text{wrap} + 1$ total clock cycles per PWM period. A wrap value of 12499 yields 12500 counter cycles.
- duty cycle is determined by comparing the slice counter against the channel level register:

$$\text{duty} = \frac{\text{level}}{\text{wrap} + 1}$$

worked example analysis (`l298n.c`):
- integer division truncation in `divider`: `divider` is declared as `uint32_t`, discarding fractional components when computing `clock_freq / (freq * 65536)`. RP2040 clock divider contains an 8-bit integer & 4-bit fractional register (1/16 precision), so declaring `divider` as `float` is necessary to pass exact fractional divisors to `pwm_set_clkdiv()`.
- 16-bit wrap overflow at 100% duty cycle: `(uint16_t)(duty_cycle * 65536)` evaluates to `65536` (`0x10000`) when `duty_cycle = 1.0f`. Casting `65536` to `uint16_t` overflows to `0`, shutting off motor drive completely when 100% output is requested.

wiring:
![L298N Pico Wiring](img/l298npico.png)
- GP2 (PWM slice 1, channel A) -> L298N ENA (remove jumper first).
- GP0 -> L298N IN3; GP1 -> L298N IN4.
- OUT3 & OUT4 -> DC motor terminals.
- GND -> L298N GND (common ground required between Pico W & external motor battery).
- logic table: GP0 HIGH & GP1 LOW rotates motor clockwise; GP0 LOW & GP1 HIGH rotates motor counterclockwise; GP0 & GP1 equal (both LOW or both HIGH) stops motor.

build & flash to Pico W:

macOS:
```sh
# copy sdk import helper from ~/pico
cp ~/pico/pico-sdk/external/pico_sdk_import.cmake .
# copy CMakeLists from bughunt & patch target to l298n
cp bughunt/CMakeLists.txt CMakeLists.txt
sed -i '' 's/bughunt4\.c/l298n.c/g; s/bughunt4/l298n/g; /algo\.c/d' CMakeLists.txt
# configure & build
mkdir -p build && cd build
cmake -DPICO_BOARD=pico_w ..
make -j8 l298n
# flash to pico (bootsel mode)
cp l298n.uf2 /Volumes/RPI-RP2
# open serial monitor (Ctrl-A Ctrl-\ to exit)
screen /dev/tty.usbmodem* 115200
```

windows (powershell):
```powershell
# copy sdk import helper & CMakeLists from bughunt
Copy-Item C:\pico\pico-sdk\external\pico_sdk_import.cmake .
Copy-Item bughunt\CMakeLists.txt CMakeLists.txt
(Get-Content CMakeLists.txt) -replace 'bughunt4\.c', 'l298n.c' -replace 'bughunt4', 'l298n' -replace '\s*algo\.c', '' | Set-Content CMakeLists.txt
# configure & build
New-Item -ItemType Directory -Force -Path build
cd build
cmake -DPICO_SDK_PATH="C:\pico\pico-sdk" ..
cmake --build . --target l298n
# flash to pico
Copy-Item l298n.uf2 -Destination D:\
```

---

## task 2: Analog-to-Digital Converter & IR Line Sensor

sample continuous analog voltages using RP2040 12-bit ADC & interface reflective IR line sensor.

extra hardware:
- IR reflective line sensor module
- 3x jumper wires

ADC hardware architecture:
- 12-bit Successive Approximation Register (SAR) ADC: converts analog voltages into digital counts from 0 to 4095 ($2^{12} - 1$). Voltage calculation with $V_{\text{REF}} = 3.3\text{ V}$ is given by:

$$V = \text{raw} \times \frac{3.3\text{ V}}{4096}$$

- 5 input channels available on RP2040: channels 0-3 are routed externally to GP26, GP27, GP28, & GP29; channel 4 connects internally to the onboard temperature sensor ($T = 27 - \frac{V_{\text{ADC}} - 0.706}{0.001721}$). Selecting channels beyond 0-4 is invalid.
- FIFO capture pipeline: `adc_fifo_setup()` enables hardware buffer capture, `adc_run(true)` streams samples into FIFO at programmed sample rates, `adc_fifo_get_blocking()` drains samples into RAM, & `adc_fifo_drain()` clears residue.

IR line sensor configuration:
![IR Line Sensor Wiring](img/linepico.png)
- reflective sensor contains an infrared emitter LED & phototransistor. Dark/black surfaces absorb IR light (low phototransistor conduction, high voltage), while reflective/white surfaces reflect IR light (high phototransistor conduction, low voltage).
- analog output (A0/OUT) connects to GP26 (ADC channel 0), VCC connects to 3.3V (pin 36), & GND connects to GND (pin 38). Digital output (D0) with threshold potentiometer is bypassed to capture full continuous analog dynamic range.

console interface commands (`adc_console`):
- `c[n]`: select ADC input channel `n` (0-3 for GP26-GP29, 4 for internal temperature sensor).
- `s`: take single conversion sample & print raw hex + calculated voltage.
- `S`: capture multi-sample batch into FIFO buffer (`N_SAMPLES`) & dump contents over USB serial.

build & flash:

macOS:
```sh
# clone or navigate to pico-examples adc_console
cd ~/pico/pico-examples/adc/adc_console
mkdir -p build && cd build
cmake -DPICO_BOARD=pico_w ..
make -j8 adc_console
# flash to pico (bootsel mode)
cp adc_console.uf2 /Volumes/RPI-RP2
# open serial monitor
screen /dev/tty.usbmodem* 115200
```

windows (powershell):
```powershell
# navigate to pico-examples adc_console
cd C:\pico\pico-examples\adc\adc_console
New-Item -ItemType Directory -Force -Path build
cd build
cmake -DPICO_SDK_PATH="C:\pico\pico-sdk" ..
cmake --build . --target adc_console
# flash to pico
Copy-Item adc_console.uf2 -Destination D:\
```

---

## task 3: PWM-to-ADC Sampling & Aliasing Analysis

loop back hardware PWM into ADC channel to evaluate Nyquist criteria & digital aliasing dynamics.

extra hardware:
- 1x jumper wire

wiring & waveform capture:
![PWM to ADC Waveform](img/ex4.png)
- connect GP0 (PWM output: 20 Hz, 50% duty cycle) directly to GP26 (ADC input channel 0) using a 1x male-male jumper wire.
- periodic timer interrupt invokes ADC sampling callback at designated intervals to reconstruct square waveform over USB serial.

aliasing analysis & sample rate comparison:

| step | sample interval | sample rate | observed behavior & mechanism |
|---|---|---|---|
| 1 | 25 ms | 40 Hz | critical Nyquist rate ($2 \times f_{\text{fundamental}}$). Sampling captures only 2 points per 20 Hz period. Waveform reconstruction exhibits severe phase dependency (resetting board shifts sample phase, altering displayed shape) & harmonic folding. 20 Hz square wave harmonics (60 Hz, 100 Hz, 140 Hz, ...) lie above 20 Hz Nyquist cutoff and fold back into baseband as phantom frequencies. |
| 2 | 5 ms | 200 Hz | oversampling ($10 \times f_{\text{fundamental}}$). 10 samples per 20 Hz period accurately resolve rising edges, flat tops, & falling edges, preserving higher harmonic fidelity without baseband foldback. |
| 3 | - | - | digital filtering cannot remove aliased frequencies post-sampling. Sampling rate selection & analog anti-aliasing pre-filtering are fundamental system design constraints. |

key insight:
- sampling rate is a fundamental architectural decision. When sampled below the Nyquist rate of the highest harmonic component, high-frequency energy folds permanently into the passband & cannot be filtered out digitally.

---

## task 4: Bug Hunt #4

```sh
cd bughunt
gcc -Wall -Wextra -o bughunt4_host bughunt4_host.c algo.c && ./bughunt4_host
```

refer to bughunt readme for more.
