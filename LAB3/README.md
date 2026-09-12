# LAB 3: Interrupts & Timers

hardware interrupts, periodic timers, IR encoder pulse measurement, ultrasonic ranging & debounced stopwatch on Raspberry Pi Pico W.

---

## hardware overview

Raspberry Pi Pico W & peripheral sensor components.

![Raspberry Pi Pico W Pinout](https://www.raspberrypi.com/documentation/microcontrollers/images/picow-pinout.svg)
- Raspberry Pi Pico W: RP2040 microcontroller, dual-core ARM Cortex-M0+, 264 KB SRAM, 26 multifunction GPIO pins (3.3V logic) & CYW43439 wireless chip.

![IR Wheel Encoder Module](img/irwheelencoder.PNG)
- HC020K optical slot sensor: 3.3V VCC, GND & OUT phototransistor signal generating digital square pulses from slotted rotating disc.

![HC-SR04P Ultrasonic Sensor](img/ultrasonic.png)
- HC-SR04P ultrasonic distance sensor: 3.3V compatible sensor. Note that standard HC-SR04 requires 5V logic & will damage RP2040 3.3V GPIO inputs.

---

## environment setup

install build toolchain & configure shell environment variables.

macOS:
```sh
# install build tools, serial monitor & ARM GCC toolchain
brew install cmake libusb
brew install --cask gcc-arm-embedded
# export environment variables permanently
echo 'export PICO_SDK_PATH=$HOME/pico/pico-sdk' >> ~/.zshrc
echo 'export PICO_BOARD=pico_w' >> ~/.zshrc
# reload shell configuration
source ~/.zshrc
```

windows (powershell):
```powershell
# set permanent user environment variables
[Environment]::SetEnvironmentVariable("PICO_SDK_PATH", "C:\pico\pico-sdk", "User")
[Environment]::SetEnvironmentVariable("PICO_BOARD", "pico_w", "User")
```

---

## task 1: GPIO Hardware Interrupts

configure event-triggered hardware interrupts using SDK callback mechanism.

extra hardware:
- 1x jumper wire

polling vs interrupts trade-off:
- polling continuously queries GPIO pin level in a tight loop. While simple, it wastes CPU cycles, increases power consumption & risks missing short asynchronous pulses if the polling interval is too long.
- hardware interrupts alert the RP2040 core immediately upon an electrical transition. This allows the processor to enter low-power sleep states (WFI/WFE) & respond asynchronously with minimal latency.

callback mechanism:
- `gpio_set_irq_enabled_with_callback(2, GPIO_IRQ_EDGE_FALL, true, &gpio_callback)` passes a function pointer to the SDK interrupt dispatcher.
- callback signature: `typedef void (*gpio_irq_callback_t)(uint gpio, uint32_t events);`. Any function matching this exact signature can be registered as an ISR.
- the compiler cannot see asynchronous SDK hardware callback invocations in the control flow graph, requiring `volatile` qualification for variables shared between ISR & main thread.

edge vs level triggering:
- `GPIO_IRQ_EDGE_FALL` & `GPIO_IRQ_EDGE_RISE` fire once per electrical signal transition.
- `GPIO_IRQ_LEVEL_LOW` & `GPIO_IRQ_LEVEL_HIGH` fire continuously as long as the pin condition persists. Switching to `GPIO_IRQ_LEVEL_LOW` without clearing the external condition causes the ISR to re-trigger endlessly, starving the main thread & hanging execution.

build & flash (`hello_gpio_irq`):
macOS:
```sh
# copy SDK import helper from ~/pico
cp ~/pico/pico-sdk/external/pico_sdk_import.cmake .
# copy hello_gpio_irq.c from pico examples
cp ~/pico/pico-examples/gpio/hello_gpio_irq/hello_gpio_irq.c .
# copy CMakeLists from bughunt & patch target to hello_gpio_irq
cp bughunt/CMakeLists.txt CMakeLists.txt
sed -i '' 's/bughunt3\.c/hello_gpio_irq.c/g; s/bughunt3/hello_gpio_irq/g' CMakeLists.txt
# build target
mkdir -p build && cd build
cmake -DPICO_BOARD=pico_w ..
make -j8 hello_gpio_irq
# flash to Pico W (bootsel mode)
cp hello_gpio_irq.uf2 /Volumes/RPI-RP2
# open serial monitor
screen /dev/tty.usbmodem* 115200
```

windows (powershell):
```powershell
# copy SDK import helper & CMakeLists from bughunt
Copy-Item C:\pico\pico-sdk\external\pico_sdk_import.cmake .
Copy-Item C:\pico\pico-examples\gpio\hello_gpio_irq\hello_gpio_irq.c .
Copy-Item bughunt\CMakeLists.txt CMakeLists.txt
(Get-Content CMakeLists.txt) -replace 'bughunt3\.c', 'hello_gpio_irq.c' -replace 'bughunt3', 'hello_gpio_irq' | Set-Content CMakeLists.txt
# build target
New-Item -ItemType Directory -Force -Path build
cd build
cmake -DPICO_SDK_PATH="C:\pico\pico-sdk" ..
cmake --build . --target hello_gpio_irq
# flash to Pico W (bootsel mode)
Copy-Item hello_gpio_irq.uf2 -Destination D:\
```

---

## task 2: IR Wheel Encoder

interface HC020K optical slot sensor on GP02 to measure pulse duration & notch counts.

extra hardware:
- HC020K optical slot sensor module & slotted disc
- 3x jumper wires

![IR Wheel Encoder Module](img/irwheelencoder.PNG)
![Encoder Waveform](img/encoder.png)

working principle:
- slotted wheel rotates between infrared LED & phototransistor detector, producing square-wave pulses as slots interrupt the optical beam.
- distance measurement: count falling/rising edges inside ISR (each detected edge represents 1 slot).
- speed measurement: measure period between successive slot starts (`previous_start_us` to `now`) using `time_us_32()` or `time_us_64()`. Rotational speed in RPM is calculated as `RPM = (60,000,000) / (period_us * SLOTS_PER_REV)`. Pulse width alone measures slot duty cycle rather than full rotational period.

![IR Pico Wiring](img/irpico.png)

wiring:
- HC020K VCC -> 3V3 (pin 36)
- HC020K GND -> GND (pin 38)
- HC020K OUT -> GP02 (pin 4)

---

## task 3: Hardware Timers & Periodic Alarms

configure RP2040 hardware timer peripherals for periodic & single-shot alarms.

RP2040 timer features:
- 64-bit microsecond counter accessible via `time_us_32()` (lower 32 bits) & `time_us_64()` (full 64 bits).
- 4 independent hardware alarms (ALARM0 to ALARM3) generating interrupts on compare match.
- timer modes: free-running (continuous counting), periodic (repeating interval interrupts) & one-shot (single delay callback).
- input capture records timestamps upon GPIO transitions, while output compare triggers actions (GPIO toggling, PWM updates) at target timer counts.

periodic alarm callbacks:
- `add_repeating_timer_ms(-delay_ms, &timer_callback, NULL, &timer)`: negative delay specifies deterministic target-to-target execution intervals independent of callback execution duration.
- positive delay schedules subsequent alarms relative to the return time of the callback.

![Periodic Timer Timing](img/periodic.png)

---

## task 4: Ultrasonic Distance Sensor

analyze HC-SR04P ranging sequence, naive blocking pitfalls & fix 3 planted defects in `ultrasonic.c`.

extra hardware:
- HC-SR04P 3.3V ultrasonic sensor module
- 4x jumper wires

![HC-SR04P Ultrasonic](img/ultrasonic.png)
![Distance Formula](img/distance.png)
![HC-SR04P Pico Wiring](img/hcsr04pico.png)

sensor operation:
1. trigger pulse: host microcontroller asserts 10 µs high pulse on TRIG pin.
2. acoustic burst: sensor emits 8-cycle 40 kHz ultrasonic burst.
3. echo pulse: ECHO pin asserts high for the duration of acoustic round-trip travel time.
4. distance conversion: `distance_cm = (pulse_us * 0.0343 cm/us) / 2 = pulse_us / 58`.

naive blocking loop vs non-blocking timer capture:
- `ultrasonic.c` uses tight `while (gpio_get(echo_pin) == 0)` & `while (gpio_get(echo_pin) == 1)` loops. For an obstacle 2 m away, round-trip travel time is 4 m (~11.66 ms) of dead CPU spinning.
- if an echo is lost or sensor is disconnected, the CPU hangs permanently in the blocking loop.
- non-blocking timer capture uses edge interrupts to capture timestamps asynchronously, leaving the CPU available for other tasks.

defects to fix in `ultrasonic.c` (3 total):
1. line 42: missing semicolon `;` after variable declaration `absolute_time_t start_time, end_time`. Compiler reports syntax error on subsequent line 44.
2. line 45: TRIG pin remains asserted high after `sleep_us(10)`. Pull TRIG low before waiting for ECHO pulse.
3. line 70: missing factor of 2 in distance division. Sound velocity of 29.1 µs/cm applies to round-trip travel; return `pulse_us / 58` (or `pulse_us / (29 * 2)`) to compute 1-way distance.

build & flash (`ultrasonic`):
macOS:
```sh
# copy CMakeLists from bughunt & patch target to ultrasonic
cp bughunt/CMakeLists.txt CMakeLists.txt
sed -i '' 's/bughunt3\.c/ultrasonic.c/g; s/bughunt3/ultrasonic/g' CMakeLists.txt
# build target
mkdir -p build && cd build
cmake -DPICO_BOARD=pico_w ..
make -j8 ultrasonic
# flash to Pico W (bootsel mode)
cp ultrasonic.uf2 /Volumes/RPI-RP2
# monitor serial output
screen /dev/tty.usbmodem* 115200
```

windows (powershell):
```powershell
# copy CMakeLists from bughunt & patch target to ultrasonic
Copy-Item bughunt\CMakeLists.txt CMakeLists.txt
(Get-Content CMakeLists.txt) -replace 'bughunt3\.c', 'ultrasonic.c' -replace 'bughunt3', 'ultrasonic' | Set-Content CMakeLists.txt
# build target
New-Item -ItemType Directory -Force -Path build
cd build
cmake -DPICO_SDK_PATH="C:\pico\pico-sdk" ..
cmake --build . --target ultrasonic
# flash to Pico W (bootsel mode)
Copy-Item ultrasonic.uf2 -Destination D:\
```

---

## task 5: GP21 Debounced Stopwatch

implement debounced stopwatch application using hardware timer interrupts & GP21 push button.

requirements:
- press & hold GP21 button: starts stopwatch & continuously prints elapsed seconds to serial monitor.
- release GP21 button: stops stopwatch & resets displayed elapsed time to 0.
- GP21 button input must be debounced using timestamp comparison to filter contact bounce.
- timekeeping must be driven by repeating hardware timer interrupt (`add_repeating_timer_ms`).

wiring:
- GP21 (pin 27) -> push button terminal 1; push button terminal 2 -> GND (internal pull-up enabled).

---

## task 6: Bug Hunt #3

extra hardware:
- HC020K optical slot sensor module & slotted disc
- 3x jumper wires

refer to bughunt readme for more.
