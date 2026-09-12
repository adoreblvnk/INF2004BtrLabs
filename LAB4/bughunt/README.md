# BUG HUNT #4: Arithmetic & Signal Chain

fix 8 arithmetic & hardware defects across `algo.c` & `bughunt4.c`.

---

## test on host first

validate algorithm logic locally on laptop before compiling for microcontroller.

```sh
# compile host test harness
gcc -Wall -Wextra -o bughunt4_host bughunt4_host.c algo.c
# run tests
./bughunt4_host
```

---

## synthetic step input analysis

debugging against physical sensors is inefficient due to noise, unrepeatable readings, & unknown ground-truth values. Feeding the algorithm a mathematically defined synthetic step input exposes arithmetic failures immediately.

- rising step (0 -> 3000): filter must climb monotonically toward target and settle within quantisation tolerance ($\pm 15$ counts for $\alpha = 1/16$).
- falling step (3000 -> 500): filter must decay smoothly toward 500. Unsigned subtraction underflow causes $(x - y)$ to wrap to $2^{32} - 2500 \approx 4.29 \times 10^9$, causing filter state to explode upward instead of decaying.

---

## defects to fix (8 total)

defects in `algo.c` (5 total):
1. line 5 (`adc_to_mv`): 16-bit intermediate calculation overflow. Multiplying `raw * VREF_MV` ($4095 \times 3300 = 13,513,500$) overflows `uint16_t` (max 65535) before division occurs. Cast `raw` to `uint32_t` before multiplying.
2. line 11 (`duty_from_adc`): fixed-point scaling integer division truncation. Dividing `raw / ADC_MAX` before multiplication truncates to 0 for all $\text{raw} < 4095$, forcing duty cycle to 0 across the entire input range except at full scale. Multiply `raw` by `wrap` in 32-bit precision before dividing by `ADC_MAX`.
3. line 14 & 24 (`iir_reset` & `iir_step`): shared channel state across inputs. A single static accumulator `y` is shared across all channels, causing alternating ADC channel samples to corrupt each other's filter state. Replace scalar with an array indexed by channel number & reset all channel accumulators in `iir_reset`.
4. line 24 (`iir_step`): filter unsigned subtraction underflow on falling step. When $x < y$ (falling step), unsigned subtraction underflows in 32-bit arithmetic, producing large positive values that blow up the filter. Cast operands to signed 32-bit integers (`int32_t`) and compute signed division.
5. line 30 (`pwm_wrap_for_hz`): duty cycle & PWM wrap off-by-one arithmetic. RP2040 PWM counter runs from 0 to `wrap` inclusive, requiring $\text{wrap} + 1$ clock ticks per cycle. Subtract 1 from the calculated clock count to produce the exact target frequency.

defects in `bughunt4.c` (3 total):
6. lines 38-39 (`sample_cb`): ADC channel sequencing inversion. `adc_read()` is called before `adc_select_input(0)`, causing channel A to read data from whatever channel was previously selected (channel B from the prior cycle) before switching. Call `adc_select_input(0)` prior to `adc_read()`.
7. line 50 & 52 (`sample_cb`): unsigned boundary comparison underflow. `delta` is declared as unsigned `uint32_t`, making `delta < 0` impossible (evaluates to false) and underflowing to large positive values when $\text{mv} > \text{TARGET\_MV}$. Declare `delta` as signed `int32_t` and handle both positive and negative errors.
8. line 59-60 (`sample_cb`): printf conversion specifier mismatch. Floating-point variable `v` is passed to `%d` format specifier, printing corrupt integer garbage due to variadic argument type promotion & register layout mismatches. Use `%f` or `%.3f` format specifiers.

---

## build & flash to Pico W

macOS:
```sh
# copy sdk import helper
cp ~/pico/pico-sdk/external/pico_sdk_import.cmake .
# configure and build
mkdir -p build && cd build
cmake -DPICO_BOARD=pico_w ..
make -j8 bughunt4
# flash to Pico W (bootsel mode)
cp bughunt4.uf2 /Volumes/RPI-RP2
# open serial monitor (Ctrl-A Ctrl-\ to exit)
screen /dev/tty.usbmodem* 115200
```

windows (powershell):
```powershell
# copy sdk import helper
Copy-Item C:\pico\pico-sdk\external\pico_sdk_import.cmake .
# configure and build
New-Item -ItemType Directory -Force -Path build
cd build
cmake -DPICO_SDK_PATH="C:\pico\pico-sdk" ..
cmake --build . --target bughunt4
# flash to Pico W (bootsel mode)
Copy-Item bughunt4.uf2 -Destination D:\
```

---

## expected output

```
BUG HUNT #4 - signal chain

adc_to_mv  (0..4095 -> 0..3300 mV)
  adc_to_mv(0)                   got 0            expect 0            ok
  adc_to_mv(1024)                got 825          expect 825          ok
  adc_to_mv(2048)                got 1650         expect 1650         ok
  adc_to_mv(4095)                got 3300         expect 3300         ok

duty_from_adc  (0..4095 -> 0..999)
  duty_from_adc(0,    999)       got 0            expect 0            ok
  duty_from_adc(1024, 999)       got 249          expect 249          ok
  duty_from_adc(2048, 999)       got 499          expect 499          ok
  duty_from_adc(4095, 999)       got 999          expect 999          ok

pwm_wrap_for_hz  (counter runs 0..wrap inclusive)
  pwm_wrap_for_hz(20 Hz, 250.0)  got 24999        expect 24999        ok
  pwm_wrap_for_hz(50 Hz, 100.0)  got 24999        expect 24999        ok

step response - rising 0 -> 3000
     187   1087   1699   2115   2396   2588   2718   2805   2865   2906 
  (should climb smoothly and settle near 3000)

step response - falling 3000 -> 500
    2778   2048   1554   1218    990    835    730    659    611    578 
  (should fall smoothly and settle near 500)

channel independence
  channel 0 fed a constant 1000, channel 1 fed a constant 2000
  channel 0 settles near         got 985 target 1000 (+/-15) ok
  channel 1 settles near         got 1985 target 2000 (+/-15) ok

SIGNAL CHAIN MATCHES THE SPECIFICATION  (0 failures)
```
