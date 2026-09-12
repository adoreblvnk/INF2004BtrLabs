# BUG HUNT #6: Telemetry Layer & Low-Level Debugging (pair / 2 boards)

fix 12 defects in CRC-8 table generation, LFSR test pattern generation, packet frame parser & sensor delay logic across `bughunt6.c` (hardware debugging exercises require 2 boards / pair).

extra hardware:
- Pico (flashed with `picoprobe.uf2`)
- 3x jumper wires

---

## test on host first

validate algorithmic logic locally before building for target microcontroller.

```sh
# compile host test harness across optimization levels
gcc -Wall -Wextra -O0 -o bughunt6_host bughunt6_host.c bughunt6.c && ./bughunt6_host
gcc -Wall -Wextra -O2 -o bughunt6_host bughunt6_host.c bughunt6.c && ./bughunt6_host
gcc -Wall -Wextra -O3 -o bughunt6_host bughunt6_host.c bughunt6.c && ./bughunt6_host
```

---

## defects to fix (12 total)

1. line 50 (`bughunt6.c`): `crc_table` declared with 255 entries (`uint8_t crc_table[255]`) instead of 256. Initialization loop writes index 255 past the end of the array into adjacent static memory. Allocate 256 elements.
2. line 67 (inside `crc8`): uninitialized accumulator variable `crc`. Local variable contains undefined stack data upon function entry. Initialize accumulator explicitly to `0x00`.
3. line 54 & 65 (`crc_init` & `crc8`): table readiness flag `crc_ready` is updated during `crc_init` but never checked inside `crc8`. Calling `crc8` prior to explicit initialization performs lookups against an uninitialized zero table. Trigger lazy table initialization on first use.
4. line 47 (`LFSR_TAPS` macro): incorrect 16-bit LFSR feedback polynomial mask (`0xB000u`). Fails to generate the maximal 65535-cycle sequence. Update tap mask to `0xB400u` for maximal length Galois sequence.
5. line 93 (inside `lfsr_period`): loop termination condition artificially capped at `n < 200`. The 16-bit generator cannot complete its full 65535-state cycle within 200 iterations. Expand iteration boundary to check full $2^{16} - 1$ states while retaining a runaway timeout.
6. line 106 (inside `parse_ts`): type punning via pointer cast (`*(uint32_t *)&frame[2]`). Causes an unaligned 32-bit word load on ARMv6-M (generating a HardFault exception on Cortex-M0+) & decodes little-endian rather than big-endian wire format. Assemble the 32-bit timestamp explicitly using bitwise shifts over 4 bytes.
7. lines 111 & 114 (inside `parse_frame`): inadequate frame header & buffer length validation. Minimum length check allows short frames, & parser fails to verify that payload length equals 6 while total frame length equals 9 before computing checksum. Reject frames with invalid lengths or malformed start-of-frame markers.
8. lines 128-132 (inside `format_reading`): returns pointer to local stack-allocated array `buf[32]`. The stack frame is destroyed upon function return, leaving a dangling pointer & undefined memory contents. Use persistent static storage or caller-provided destination buffers.
9. line 136 (inside `label_for`): dynamic memory allocation allocates `strlen(name)` bytes without accounting for the string null terminator. `strcpy` writes the null byte out of bounds into heap metadata. Allocate `strlen(name) + 1` bytes & verify allocation success.
10. line 147 (inside `calibration_delay`): empty loop `for (uint32_t i = 0; i < 6000; i++) ;` has no observable side-effects. Compiler dead-code elimination removes the loop entirely under `-O2` & `-O3`. Use hardware timer delay functions (`sleep_us(500)` on target or POSIX monotonic sleep on host).
11. line 154 (`conversion_done`): ISR completion flag lacks `volatile` type qualifier. The compiler assumes the flag cannot change asynchronously inside `wait_for_conversion` & optimizes the loop into an infinite register check. Mark flag `static volatile bool conversion_done`.
12. line 174 (inside `checksum_all`): loop counter `uint8_t i` overflows at 255. When summing buffers longer than 255 bytes (such as the 300-byte test buffer), the 8-bit index wraps to 0, resulting in an infinite loop. Use `uint16_t` or `size_t` for the index variable.

---

## 3 hardware debugging exercises

### exercise A: catch buffer overflow with GDB data watchpoint

locate the out-of-bounds write occurring in `crc_init` on target hardware.

```text
# start gdb session & break before table initialization
(gdb) break crc_init
(gdb) continue
# set hardware data watchpoint on the byte immediately following the table
(gdb) watch -l *(unsigned char *)((char *)&crc_table + sizeof(crc_table))
(gdb) continue
```

GDB halts execution at the exact instruction attempting to write index 255 out of bounds.

### exercise B: decode ARMv6-M HardFault on unaligned word load

Cortex-M0+ lacks unaligned memory access support present in Cortex-M3/M4 cores.
- when `parse_ts` executes `*(uint32_t *)&frame[2]`, the address is offset by 2 bytes from a 4-byte boundary.
- the processor immediately triggers a HardFault exception & stacks 8 registers: `R0, R1, R2, R3, R12, LR, PC, xPSR`.
- inspect stacked `PC` via GDB (`info registers` or examining `MSP`/`PSP` memory) & locate the faulting `LDR` instruction in disassembly (`arm-none-eabi-objdump -d bughunt6.elf`).

### exercise C: prove optimization defects from disassembly

compare emitted assembly between Debug (`-O0` / `-Og`) & Release (`-O3`) builds.

```sh
arm-none-eabi-objdump -d build-debug/bughunt6.elf > debug.asm
arm-none-eabi-objdump -d build-release/bughunt6.elf > release.asm
```

- inspect `calibration_delay`: in Debug build, loop counter decrements & branches exist; in Release build, the function collapses to a single `bx lr` return instruction.
- inspect `wait_for_conversion`: in Release build without `volatile`, the compiler loads `conversion_done` into a register once before the loop & branches to itself forever (`b .`).

---

## build & flash to Pico W

macOS:
```sh
# copy pico sdk import helper
cp ~/pico/pico-sdk/external/pico_sdk_import.cmake .
# build debug target
mkdir -p build && cd build
cmake -DPICO_BOARD=pico_w ..
make -j8 bughunt6
# flash to Pico W via bootsel
cp bughunt6.uf2 /Volumes/RPI-RP2
# open serial monitor
screen /dev/tty.usbmodem* 115200
```

windows (powershell):
```powershell
Copy-Item C:\pico\pico-sdk\external\pico_sdk_import.cmake .
New-Item -ItemType Directory -Force -Path build
cd build
cmake -DPICO_SDK_PATH="C:\pico\pico-sdk" ..
cmake --build . --target bughunt6
Copy-Item bughunt6.uf2 -Destination D:\
```

---

## expected output

```text
BUG HUNT #6 - telemetry layer

  crc8 before explicit init          got 244          expect 244          ok
crc8 - reference vector for CRC-8/ATM
  crc8("123456789", 9)               got 244          expect 244          ok
  crc8("123456789", 9) again         got 244          expect 244          ok

lfsr - must visit every non-zero 16-bit value
  lfsr_period()                      got 65535        expect 65535        ok

parse_frame
  timestamp                          got 287454020    expect 287454020    ok
  value                              got 1800         expect 1800         ok
  feeding a frame whose length byte says 200...

format_reading
  returned: "t=42 v=7"  (expect "t=42 v=7")

label_for
  returned: "temperature"  (expect "temperature")

calibration_delay - datasheet requires at least 500 us
  measured 500.00 us per call
  (this number is meaningless on your laptop - measure it on the Pico,
   at -O0 & at -O3, & explain the difference)

checksum_all
  summing 300 bytes of value 1, expecting 300...
  checksum_all(big, 300)             got 300          expect 300          ok

TELEMETRY LAYER MATCHES THE SPECIFICATION  (0 failures)
```
