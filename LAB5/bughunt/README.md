# BUG HUNT #5: FreeRTOS Sensor Pipeline

fix 9 concurrency, synchronization & configuration defects across FreeRTOS sensor pipeline in `bughunt5.c` & `FreeRTOSConfig.h`.

---

## kernel diagnostics & assertions

enable FreeRTOS diagnostic hooks to trap contract violations & stack overflows.

diagnostic configuration in `FreeRTOSConfig.h`:
- `configASSERT(x)`: macro evaluates kernel preconditions & halts CPU upon contract violation. Prevents silent memory corruption from propagating:
```c
#define configASSERT(x) if ((x) == 0) { taskDISABLE_INTERRUPTS(); for( ;; ); }
```
- `configCHECK_FOR_STACK_OVERFLOW`: set to `2` to enable stack boundary & canary verification on each context switch.
- `vApplicationStackOverflowHook(TaskHandle_t xTask, char *pcTaskName)`: hook function called automatically by kernel when stack overflow occurs.
- `uxTaskGetStackHighWaterMark(TaskHandle_t xTask)`: returns minimum remaining stack headroom in 32-bit words since task creation.
- `configUSE_MALLOC_FAILED_HOOK`: traps heap allocation failure when dynamic memory is exhausted.

debugging concurrency:
- serial logging (`printf`) alters task execution timing, masks race conditions & distorts real-time behaviour. Kernel assertions & high-water mark introspection provide non-invasive diagnostics.

---

## defects to fix (9 total)

1. line 88 (`FreeRTOSConfig.h`): unasserted kernel contracts. `configASSERT(x)` is defined as an empty macro, silently discarding kernel contract & precondition checks. Define assertion trap to disable interrupts & halt execution on failure (`configCHECK_FOR_STACK_OVERFLOW` & `configUSE_MALLOC_FAILED_HOOK` are also disabled).
2. line 122 (`bughunt5.c`): `sizeof` on pointer vs buffer. `sizeof(&sample)` passes pointer size (4 bytes on 32-bit RP2040) instead of 16-bit integer size (2 bytes) to `xMessageBufferSend()`. The receiver buffer (2 bytes) cannot fit the 4-byte message, causing `xMessageBufferReceive()` to return 0. Pass `sizeof(sample)`.
3. line 124 (`bughunt5.c`): raw tick delay instead of time macro & drift accumulation. `vTaskDelay(1)` delays for 1 raw tick rather than converting `SAMPLE_PERIOD_MS` via `pdMS_TO_TICKS()`. Replace with `xTaskDelayUntil()` to eliminate cumulative loop execution drift.
4. lines 38, 134, 152 (`bughunt5.c`): multiple tasks reading 1 Message Buffer. Both `avg_task` & `mean_task` call `xMessageBufferReceive()` on 1 message buffer `mbuf`. FreeRTOS Message Buffers strictly require 1 reader (SPSC). Use dedicated message buffers or queues to fan out samples to multiple consumers.
5. lines 63-70, 137, 155 (`bughunt5.c`): unprotected shared global ring buffer. Both `avg_task` & `mean_task` call `ring_push()` on shared global ring without synchronization, corrupting head/tail indices & distorting moving average window. Grant exclusive ring buffer ownership to `avg_task` & remove `ring_push()` from `mean_task`.
6. lines 84-92, 140, 157 (`bughunt5.c`): non-reentrant `static` variables. `running_mean()` contains static accumulator & counter variables shared across callers, causing `avg_task` & `mean_task` to corrupt each other's calculations. Maintain independent accumulator & count state per task.
7. line 100 (`bughunt5.c`): unhandled return value on queue send. `say()` ignores return value of `xQueueSend()`, causing silent telemetry loss when `print_q` is full. Check return status & handle full queue conditions or block with appropriate timeout.
8. line 139 (`bughunt5.c`): incorrect moving average divisor during warm-up. `avg_task` divides `ring_sum()` by constant `RING_LEN` (10) before 10 samples have been collected, distorting early averages. Divide by populated element count `fill` until buffer is full.
9. line 209 (`bughunt5.c`): stack exhaustion. `print_task` is allocated `configMINIMAL_STACK_SIZE` (256 words). Handling floating-point conversions in `printf` requires larger stack depth, leading to stack overflow. Increase stack allocation depth for `print_task`.

---

## build & flash to Pico W

macOS:
```sh
# copy sdk import helper
cp ~/pico/pico-sdk/external/pico_sdk_import.cmake .
# configure & build bughunt5
mkdir -p build && cd build
cmake -DPICO_BOARD=pico_w -DFREERTOS_KERNEL_PATH=$HOME/pico/FreeRTOS-Kernel ..
make -j8 bughunt5
# flash to Pico W (bootsel mode)
cp bughunt5.uf2 /Volumes/RPI-RP2
# open serial monitor (Ctrl-A Ctrl-\ to exit)
screen /dev/tty.usbmodem* 115200
```

windows (powershell):
```powershell
# copy sdk import helper
Copy-Item C:\pico\pico-sdk\external\pico_sdk_import.cmake .
# configure & build bughunt5
New-Item -ItemType Directory -Force -Path build
cd build
cmake -DPICO_SDK_PATH="C:\pico\pico-sdk" -DPICO_BOARD=pico_w -DFREERTOS_KERNEL_PATH="C:\FreeRTOS-Kernel-main" ..
cmake --build . --target bughunt5
# flash to Pico W (bootsel mode)
Copy-Item bughunt5.uf2 -Destination D:\
```

---

## expected output

```
BUG HUNT #5 - starting scheduler
moving avg (last 10): 24.5 C
running mean since boot: 24.5 C
moving avg (last 10): 24.6 C
running mean since boot: 24.5 C
-- heap free: 124832 bytes --
moving avg (last 10): 24.6 C
running mean since boot: 24.5 C
```
