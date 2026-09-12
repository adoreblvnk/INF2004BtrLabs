# BUG HUNT #1: Bit Manipulation

fix 6 defects in bit manipulation helpers across `bughunt1.c`.

---

## test on host first

validate logic locally on laptop before compiling for microcontroller.

```sh
# compile host test harness
gcc -Wall -Wextra -o bughunt1 bughunt1.c
# run tests
./bughunt1
```

---

## defects to fix (6 total)

1. line 16 (`bughunt1.c`): missing header for standard fixed-width integer types (`uint8_t`, `uint32_t`). Include `<stdint.h>`.
2. line 32 (inside `count_bits`): missing statement terminator `;` after variable declaration.
3. line 30 (signature of `count_bits`): signed `int` parameter causes arithmetic sign extension on negative inputs (e.g. `count_bits(-1)` shifts `1`s from the left indefinitely). Change parameter type to an unsigned 32-bit integer.
4. line 51 (inside `even_parity`): missing closing brace `}` before `else` block.
5. line 64 (inside `pin_is_clear`): operator precedence issue where `==` evaluates before `&`. Add parentheses around the bitwise AND expression so it is evaluated before the equality check.
6. line 76 (inside `reverse_bits`): loop bound off-by-one. Running 33 iterations causes shifts by 32 and negative amounts (undefined behaviour). Constrain loop to run exactly 32 iterations (0 to 31).

note: `swap_nibbles` (line 87) is already correct. Do not modify working functions.

---

## build & flash to Pico W

macOS:
```sh
# copy sdk import helper
cp ~/pico/pico-sdk/external/pico_sdk_import.cmake .
# configure and build
mkdir -p build && cd build
cmake -DPICO_BOARD=pico_w ..
make -j8 bughunt1
# flash to pico (bootsel mode)
cp bughunt1.uf2 /Volumes/RPI-RP2
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
cmake --build . --target bughunt1
# flash to pico
Copy-Item bughunt1.uf2 -Destination D:\
```

---

## expected output

```
BUG HUNT #1 - bit manipulation

count_bits
  count_bits(0x000000FF)             got 8          expect 8          ok
  count_bits(0x00000024)             got 2          expect 2          ok
  count_bits(-1)                     got 32         expect 32         ok
even_parity
  even_parity(0x000000FF)            got 1          expect 1          ok
  even_parity(0x00000007)            got 0          expect 0          ok
pin_is_clear (mask 0x24)
  pin_is_clear(LED_MASK, 2)          got 0          expect 0          ok
  pin_is_clear(LED_MASK, 3)          got 1          expect 1          ok
  pin_is_clear(LED_MASK, 5)          got 0          expect 0          ok
reverse_bits
  reverse_bits(0x00000001)           got 0x80000000  expect 0x80000000  ok
  reverse_bits(0x12345678)           got 0x1E6A2C48  expect 0x1E6A2C48  ok
  reverse_bits(0xFFFFFFFF)           got 0xFFFFFFFF  expect 0xFFFFFFFF  ok
swap_nibbles
  swap_nibbles(0x12345678)           got 0x21436587  expect 0x21436587  ok

ALL TESTS PASS  (0 failures)
```
