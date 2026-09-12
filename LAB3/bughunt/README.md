# BUG HUNT #3: Interrupt State & Timing

investigate compiler optimization side effects, concurrency races, timer rollover & ISR latency in `bughunt3.c`.

extra hardware:
- HC020K optical slot sensor module & slotted disc
- 3x jumper wires

---

## debug vs release optimization analysis

compiler optimization flags alter code generation & memory access patterns for shared variables.

build comparison:
macOS:
```sh
# build Debug (-O0)
mkdir -p build-debug && cd build-debug
cmake -DPICO_BOARD=pico_w -DCMAKE_BUILD_TYPE=Debug -DPICO_DEOPTIMIZED_DEBUG=1 ..
make -j8 bughunt3
cd ..

# build Release (-O3)
mkdir -p build-release && cd build-release
cmake -DPICO_BOARD=pico_w -DCMAKE_BUILD_TYPE=Release ..
make -j8 bughunt3
cd ..
```

windows (powershell):
```powershell
# build Debug (-O0)
New-Item -ItemType Directory -Force -Path build-debug
cd build-debug
cmake -DPICO_SDK_PATH="C:\pico\pico-sdk" -DCMAKE_BUILD_TYPE=Debug -DPICO_DEOPTIMIZED_DEBUG=1 ..
cmake --build . --target bughunt3
cd ..

# build Release (-O3)
New-Item -ItemType Directory -Force -Path build-release
cd build-release
cmake -DPICO_SDK_PATH="C:\pico\pico-sdk" -DCMAKE_BUILD_TYPE=Release ..
cmake --build . --target bughunt3
cd ..
```

disassembly inspection:
```sh
# dump disassembly from compiled ELF binaries
arm-none-eabi-objdump -d build-debug/bughunt3.elf > debug.asm
arm-none-eabi-objdump -d build-release/bughunt3.elf > release.asm
```

disassembly difference in main wait loop (`while (pulse_count < TARGET_SLOTS)`):
- Debug (`-O0` / `-DPICO_DEOPTIMIZED_DEBUG=1`): `pulse_count` is reloaded from SRAM on every single loop iteration:
```asm
100003f8:  ldr   r3, [r3, #0]     ; read pulse_count from memory
100003fc:  cmp   r3, #19          ; compare with TARGET_SLOTS - 1
100003fe:  bls.n 100003f8         ; branch back and reload
```
- Release (`-O3`): compiler performs loop-invariant code motion because `pulse_count` is not marked `volatile`. Having proven that single-threaded `main()` does not modify `pulse_count`, the compiler hoists the load before the loop & generates an infinite self-branch:
```asm
100003de:  ldr   r3, [r6, #0]     ; read pulse_count once before loop
100003e0:  cmp   r3, #19          ; check initial count
100003e2:  bhi.n 100003e6         ; exit if already completed
100003e4:  b.n   100003e4         ; infinite branch to self (hangs forever)
```
- mechanism: the compiler has no visibility into asynchronous hardware ISR invocations. Declaring shared state variables `volatile` enforces memory reloads on every access.

---

## defects to fix (8 total)

1. lines 30-37 (`bughunt3.c`): shared state variables (`pulse_count`, `last_edge_us`, `slot_width_us`, `slot_period_us`, `state`) lack `volatile` qualification. Declare all variables written by ISR & read by `main()` as `volatile` to prevent compiler register caching in Release mode (`-O3`).
2. line 106 (`bughunt3.c`): non-atomic 64-bit access on 32-bit ARM Cortex-M0+. Reading 64-bit variable `last_edge_us` requires 2 32-bit load instructions (`ldr`), risking word tearing if an ISR fires between the 2 loads. Protect multi-word reads in `main()` by disabling interrupts with `save_and_disable_interrupts()` & restoring with `restore_interrupts()`. Note that `volatile` prevents register caching but does not guarantee atomicity.
3. line 52 (`bughunt3.c`): unsigned addition overflow in debounce comparison (`now > last_debounce_us + DEBOUNCE_US`). When `last_debounce_us` approaches `0xFFFFFFFF`, the addition wraps around to a small number, causing premature trigger or failure. Use unsigned subtraction (`(uint32_t)(now - last_debounce_us) > DEBOUNCE_US`).
4. line 26 (`bughunt3.c`): excessive debounce duration (`#define DEBOUNCE_US 50000u` = 50 ms). At realistic wheel speeds (e.g. 60 RPM with 20 slots = 50 ms slot period, pulse width ~25 ms), valid slot pulses are shorter than 50 ms & get discarded. Reduce debounce threshold (e.g. 1000 µs / 1 ms).
5. lines 57-70 (`bughunt3.c`): switch statement fallthrough. Missing `break;` statement in `case ENC_IDLE:` causes immediate fallthrough into `case ENC_SLOT:`, calculating zero slot width (`now - slot_start_us = 0`) & incrementing `pulse_count` on a single edge.
6. lines 72-73 (`bughunt3.c`): blocking `printf` inside ISR. USB serial transmission takes milliseconds & delays interrupt completion, dropping subsequent high-speed edges & distorting timing (Heisenbug). Remove `printf` from ISR & defer output reporting to `main()`.
7. lines 96-99 (`bughunt3.c`): interrupt edge configuration mismatch. Interrupt is registered only for falling edges (`GPIO_IRQ_EDGE_FALL`), but measuring slot pulse width requires capturing both slot entry (falling) & slot exit (rising) edges (`GPIO_IRQ_EDGE_FALL | GPIO_IRQ_EDGE_RISE`).
8. line 112 (`bughunt3.c`): format string specifier mismatch. `finished_at` is a 64-bit unsigned integer (`uint64_t`), but formatted using `%u` (which expects a 32-bit integer), leading to undefined behavior & corrupted terminal output. Use `%llu` format specifier with an `(unsigned long long)` cast.

---

## ISR instrumentation techniques

efficient techniques for observing ISR behavior without timing distortion.

GPIO probe pin toggle:
- toggling a GPIO pin takes ~2 CPU cycles (~30 ns at 125 MHz) compared to milliseconds for serial `printf`.

```c
#define PROBE_PIN 15
// in main before enabling IRQ:
// gpio_init(PROBE_PIN); gpio_set_dir(PROBE_PIN, GPIO_OUT);

void encoder_isr(uint gpio, uint32_t events) {
    gpio_put(PROBE_PIN, 1);    // probe entry (~30 ns)
    // ISR logic
    gpio_put(PROBE_PIN, 0);    // probe exit
}
```

ring buffer logging:
- record event timestamps into a lightweight circular buffer inside ISR & drain/print from `main()` asynchronously:

```c
static volatile uint32_t log_buf[64];
static volatile uint8_t  log_head = 0;

// inside ISR:
log_buf[log_head++ & 63] = slot_width_us;

// in main():
// drain and printf at background priority
```

---

## build & flash to Pico W

macOS:
```sh
# copy SDK import helper
cp ~/pico/pico-sdk/external/pico_sdk_import.cmake .
# build target
mkdir -p build && cd build
cmake -DPICO_BOARD=pico_w ..
make -j8 bughunt3
# flash to Pico W (bootsel mode)
cp bughunt3.uf2 /Volumes/RPI-RP2
# open serial monitor (Ctrl-A Ctrl-\ to exit)
screen /dev/tty.usbmodem* 115200
```

windows (powershell):
```powershell
# copy SDK import helper
Copy-Item C:\pico\pico-sdk\external\pico_sdk_import.cmake .
# configure and build
New-Item -ItemType Directory -Force -Path build
cd build
cmake -DPICO_SDK_PATH="C:\pico\pico-sdk" ..
cmake --build . --target bughunt3
# flash to Pico W (bootsel mode)
Copy-Item bughunt3.uf2 -Destination D:\
```
