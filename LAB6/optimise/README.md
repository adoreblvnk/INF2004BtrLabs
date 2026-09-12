# EXERCISE: Code Optimization on RP2040

4 computationally expensive functions; implement fast equivalents preserving bit-exact numerical correctness across ARMv6-M architectural constraints.

---

## core architectural constraints (ARMv6-M / Cortex-M0+)

why host performance benchmarks diverge from RP2040 target microcontroller execution:
- no hardware floating-point unit (FPU): all `float` & `double` operations invoke software emulation routines (`__aeabi_fadd`, `__aeabi_dmul`, `__aeabi_d2uiz`). A single floating-point multiplication requires tens to hundreds of CPU cycles compared to 1 cycle on modern host x86_64 / ARM64 processors.
- no hardware divide instruction: ARMv6-M lacks `UDIV` & `SDIV` instructions. Non-constant integer divisions (`/`) & modulo operations (`%`) compile into runtime library calls (`__aeabi_uidiv`). While RP2040 includes a hardware divider in its SIO peripheral block, hardware register access remains significantly slower than single-cycle integer multiplication (`MULS`).

---

## 4 optimization pairs

summary of benchmark functions & optimization strategies:

| # | Function | Slow mechanism | Fast optimization | Architectural bottleneck |
|---|---|---|---|---|
| 1 | `mv_convert` | `double` precision arithmetic per sample | 32-bit unsigned integer scaling arithmetic | software FPU emulation calls |
| 2 | `count_in_band` | expensive non-inlined function calls inside loop | hoist invariant threshold calculations outside loop | redundant function call overhead & pipeline stalls |
| 3 | `crc8` | bitwise bit-by-bit shifting (8 iterations/byte) | 256-entry precomputed table lookup (1 access/byte) | loop branching & serial bit extraction |
| 4 | `normalise` | software integer division per buffer element | division hoisting / fixed-point reciprocal multiplication | missing ARMv6-M `UDIV` hardware instruction |

### 1. ADC millivolt conversion (`mv_convert`)

convert 12-bit ADC raw counts ($0 \dots 4095$) to millivolts ($0 \dots 3300\text{ mV}$) via integer arithmetic.
- slow implementation casts each raw count to `double`, computes `(raw[i] * 3300.0) / 4095.0`, & casts back to integer.
- fast implementation performs pure integer multiplication & division: `((uint32_t)raw[i] * 3300u) / 4095u`.
- intermediate integer range: maximum product is $4095 \times 3300 = 13,513,500$, which fits comfortably within a standard unsigned 32-bit integer (`uint32_t` maximum is $4,294,967,295$), avoiding arithmetic overflow.

### 2. loop invariant hoisting (`count_in_band`)

hoist invariant threshold computations out of the sample processing loop.
- slow implementation repeatedly calls non-inlinable calibration functions `calib_low(cal)` & `calib_high(cal)` inside every iteration of an $N$-element loop ($2N$ function calls total).
- fast implementation computes `uint16_t low = calib_low(cal);` & `uint16_t high = calib_high(cal);` exactly once prior to the loop, reducing function calls from $2N$ to 2.

### 3. table-driven CRC-8 (`crc8`)

replace bitwise polynomial division loops with a precomputed 256-entry lookup table.
- slow implementation processes data byte-by-byte with an inner loop running 8 iterations per byte (checking MSB & XORing with polynomial `0x07`).
- fast implementation initializes a 256-entry lookup table once (`static uint8_t table[256];`) & performs 1 table lookup per input byte: `crc = table[crc ^ data[i]];`.
- memory vs speed trade-off: table-driven CRC-8 requires 256 bytes of SRAM or Flash memory. In exchange, execution time drops by approximately $8\times$. On memory-constrained microcontrollers (e.g. ATtiny10 with 32 bytes of SRAM, or PIC microcontrollers with 128 bytes RAM), allocating 256 bytes for a lookup table is unacceptable & the bit-by-bit approach must be retained.

### 4. reciprocal fixed-point normalisation (`normalise`)

normalize buffer elements to permille ($0 \dots 1000$) by computing total sum once & replacing repeated divisions.
- slow implementation computes buffer sum `total`, then executes `out[i] = (raw[i] * 1000) / total` for all $N$ elements, invoking $N$ division operations.
- fast implementation computes `total` once. If `total == 0`, zeros output buffer immediately. For non-zero `total`, precomputes a 64-bit reciprocal fixed-point multiplier `uint64_t inv = ((1ULL << 32) * 1000ULL) / total;` & computes each sample using multiplication & right shift: `out[i] = (uint16_t)(((uint64_t)raw[i] * inv) >> 32);` (or leverages integer division optimization).
- numerical precision constraint: fixed-point implementations must match the integer truncation of the slow version across all test vectors without a single LSB discrepancy.

---

## test on host first

verify algorithmic correctness across all test vectors before profiling execution speed on microcontroller.

```sh
# compile host test harness
gcc -O2 -Wall -Wextra -o optimise_host optimise_host.c optimise.c
# run correctness checks
./optimise_host
```

---

## build & measure on Pico W

profile execution durations on RP2040 across 3 optimization levels (`-O0`, `-O2`, `-O3`).

macOS:
```sh
# copy pico import helper
cp ~/pico/pico-sdk/external/pico_sdk_import.cmake .
# build target at -O2
mkdir -p build-O2 && cd build-O2
cmake -DPICO_BOARD=pico_w -DCMAKE_BUILD_TYPE=Release ..
make -j8 optimise
# flash to Pico W
cp optimise.uf2 /Volumes/RPI-RP2
# monitor timing output
screen /dev/tty.usbmodem* 115200
```

windows (powershell):
```powershell
Copy-Item C:\pico\pico-sdk\external\pico_sdk_import.cmake .
New-Item -ItemType Directory -Force -Path build-O2
cd build-O2
cmake -DPICO_SDK_PATH="C:\pico\pico-sdk" -DCMAKE_BUILD_TYPE=Release ..
cmake --build . --target optimise
Copy-Item optimise.uf2 -Destination D:\
```

inspect compiler-generated disassembly:
```sh
# disassemble target binary to inspect instructions
arm-none-eabi-objdump -d build-O2/optimise.elf > O2.asm
```

---

## deliverables & reporting requirements

record the following findings in your submission:
1. benchmark timing table: execution times (in microseconds) for all 4 functions across `-O0`, `-O2` & `-O3` optimization levels on RP2040.
2. optimization mechanisms: 1 concise sentence per function explaining the exact architectural mechanism delivering the speedup.
3. disassembly analysis: 1 annotated disassembly extract proving the compiler transformation (e.g. elimination of `__aeabi_dmul` calls or hoisted loop branches).
4. memory vs speed trade-off explanation: byte cost of the CRC-8 lookup table & an example device architecture where this trade-off would be rejected.
5. compiler vs manual optimization comparison: document any instances where `-O2`/`-O3` compiler optimization closed the performance gap between slow & fast code without manual intervention.
