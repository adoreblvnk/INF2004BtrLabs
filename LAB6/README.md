# LAB 6: Optimization & Debugging

execution time measurement, clock scaling, hardware debugging with PicoProbe & OpenOCD on Raspberry Pi Pico W.

---

## hardware & wiring overview

PicoProbe debugger & target Raspberry Pi Pico W wiring.

![PicoProbe Hardware Connection](picoprobe.jpg)
- PicoProbe (left): Raspberry Pi Pico running `picoprobe.uf2` firmware acting as CMSIS-DAP debugger & UART bridge.
- target Pico W (right): RP2040 microcontroller executing debug firmware.
- SWD interface connection: GP2 (SWCLK) -> SWCLK (target pin), GP3 (SWDIO) -> SWDIO (target pin), GND -> GND.
- UART pass-through connection: GP4 (UART1 TX) -> GP1 (UART0 RX on target), GP5 (UART1 RX) -> GP0 (UART0 TX on target).

---

## environment setup

install OpenOCD, GDB & configure VSCode debug extension.

macOS:
```sh
# install openocd with rp2040 support & arm toolchain via brew
brew install openocd-rp2040 arm-none-eabi-gdb
# download picoprobe firmware binary for debugger pico
mkdir -p ~/pico/picoprobe && cd ~/pico/picoprobe
curl -LO https://github.com/raspberrypi/picoprobe/releases/latest/download/picoprobe.uf2
```

windows (powershell):
```powershell
# clone & build openocd or install via pico setup installer
New-Item -ItemType Directory -Force -Path C:\pico\picoprobe
Invoke-WebRequest -Uri "https://github.com/raspberrypi/picoprobe/releases/latest/download/picoprobe.uf2" -OutFile "C:\pico\picoprobe\picoprobe.uf2"
```

VSCode `.vscode/launch.json` configuration:
```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Pico Debug (Cortex-Debug)",
      "cwd": "${workspaceFolder}",
      "executable": "${command:cmake.launchTargetPath}",
      "request": "launch",
      "type": "cortex-debug",
      "servertype": "openocd",
      "gdbPath": "arm-none-eabi-gdb",
      "device": "RP2040",
      "configFiles": [
        "interface/cmsis-dap.cfg",
        "target/rp2040.cfg"
      ],
      "svdFile": "${env:PICO_SDK_PATH}/src/rp2040/hardware_regs/rp2040.svd",
      "runToEntryPoint": "main",
      "openOCDLaunchCommands": [
        "adapter speed 5000"
      ]
    }
  ]
}
```

---

## task 1: execution time measurement & optimization

measure execution duration accurately using `pico/time.h` APIs & prevent compiler dead-code elimination.

key timing principles:
- `absolute_time_diff_us()` returns a signed 64-bit integer (`int64_t`). Storing the result in `uint32_t` truncates the upper 32 bits, while printing with `%d` invokes undefined behaviour. Use `int64_t` & the `PRId64` format specifier from `<inttypes.h>`.
- dead-code elimination: under optimization flags (`-O2`, `-O3`), loops computing values that do not affect observable program state are deleted entirely by the compiler. Declare a `static volatile uint32_t sink;` variable to force memory writes on each iteration.

benchmark code:
```c
#include <stdio.h>
#include <inttypes.h>
#include "pico/stdlib.h"
#include "pico/time.h"

// volatile sink prevents compiler from eliminating the benchmark loop
static volatile uint32_t sink;

void timeConsumingTask(void) {
    for (int i = 0; i < 1000000; i++) {
        sink = (uint32_t)(i * 2);
    }
}

int main(void) {
    stdio_init_all();
    sleep_ms(3000);

    printf("measuring execution time...\n");

    // measure execution time before optimization
    absolute_time_t start_time = get_absolute_time();
    timeConsumingTask();
    absolute_time_t end_time = get_absolute_time();

    int64_t execution_time_before = absolute_time_diff_us(start_time, end_time);
    printf("Execution Time Before: %" PRId64 " microseconds\n", execution_time_before);

    // measure execution time after optimization
    start_time = get_absolute_time();
    timeConsumingTask();
    end_time = get_absolute_time();

    int64_t execution_time_after = absolute_time_diff_us(start_time, end_time);
    printf("Execution Time After:  %" PRId64 " microseconds\n", execution_time_after);

    return 0;
}
```

build & flash to Pico W:

macOS:
```sh
# copy SDK import helper from ~/pico
cp ~/pico/pico-sdk/external/pico_sdk_import.cmake .
# copy CMakeLists from bughunt & patch target to task1
cp bughunt/CMakeLists.txt CMakeLists.txt
sed -i '' '/bughunt6\.c/d; s/bughunt6_pico\.c/task1.c/g; s/bughunt6/task1/g' CMakeLists.txt
# configure build directory & compile
mkdir -p build && cd build
cmake -DPICO_BOARD=pico_w ..
make -j8 task1
# flash UF2 via bootsel volume
cp task1.uf2 /Volumes/RPI-RP2
# monitor serial output
screen /dev/tty.usbmodem* 115200
```

windows (powershell):
```powershell
# copy SDK import helper & CMakeLists from bughunt
Copy-Item C:\pico\pico-sdk\external\pico_sdk_import.cmake .
Copy-Item bughunt\CMakeLists.txt CMakeLists.txt
(Get-Content CMakeLists.txt) -notmatch 'bughunt6\.c' | Set-Content CMakeLists.txt
(Get-Content CMakeLists.txt) -replace 'bughunt6_pico\.c', 'task1.c' -replace 'bughunt6', 'task1' | Set-Content CMakeLists.txt
# build
New-Item -ItemType Directory -Force -Path build
cd build
cmake -DPICO_SDK_PATH="C:\pico\pico-sdk" ..
cmake --build . --target task1
# flash
Copy-Item task1.uf2 -Destination D:\
```

---

## task 2: power reduction via clock frequency

scale RP2040 system clock down from default 125 MHz to 48 MHz to reduce dynamic power consumption.

key clock management concepts:
- dynamic power consumption in CMOS digital logic is proportional to frequency ($P \propto C \cdot V^2 \cdot f$). Lowering system clock frequency reduces power consumption linearly at the expense of computational throughput.
- reducing system clock impacts all peripherals clocked from `clk_sys` & `clk_peri` (UART baud rate generators, PWM dividers, timer ticks). Peripheral dividers must be updated accordingly.
- the `hello_48MHz` sample illustrates calling `set_sys_clock_48mhz()` before peripheral initialization.

```c
#include <stdio.h>
#include "pico/stdlib.h"
#include "hardware/clocks.h"

int main(void) {
    // configure system clock to 48 MHz
    set_sys_clock_48mhz();

    // initialize stdio after clock change so UART baud rate calculates correctly
    stdio_init_all();

    while (true) {
        printf("hello 48 MHz clock: current sys_clk = %lu Hz\n", clock_get_hz(clk_sys));
        sleep_ms(1000);
    }
    return 0;
}
```

build & flash (`hello_48MHz`):

macOS:
```sh
# copy hello_48MHz.c from pico examples
cp ~/pico/pico-examples/clocks/hello_48MHz/hello_48MHz.c .
# copy CMakeLists from bughunt & patch target to hello_48MHz
cp bughunt/CMakeLists.txt CMakeLists.txt
sed -i '' '/bughunt6\.c/d; s/bughunt6_pico\.c/hello_48MHz.c/g; s/bughunt6/hello_48MHz/g; s/pico_stdlib/pico_stdlib hardware_clocks/g' CMakeLists.txt
# build & flash
mkdir -p build && cd build
cmake -DPICO_BOARD=pico_w ..
make -j8 hello_48MHz
cp hello_48MHz.uf2 /Volumes/RPI-RP2
```

windows (powershell):
```powershell
# copy hello_48MHz.c from pico examples
Copy-Item C:\pico\pico-examples\clocks\hello_48MHz\hello_48MHz.c .
# copy CMakeLists from bughunt & patch target to hello_48MHz
Copy-Item bughunt\CMakeLists.txt CMakeLists.txt
(Get-Content CMakeLists.txt) -notmatch 'bughunt6\.c' | Set-Content CMakeLists.txt
(Get-Content CMakeLists.txt) -replace 'bughunt6_pico\.c', 'hello_48MHz.c' -replace 'bughunt6', 'hello_48MHz' -replace 'pico_stdlib', 'pico_stdlib hardware_clocks' | Set-Content CMakeLists.txt
# build & flash
New-Item -ItemType Directory -Force -Path build
cd build
cmake -DPICO_SDK_PATH="C:\pico\pico-sdk" ..
cmake --build . --target hello_48MHz
Copy-Item hello_48MHz.uf2 -Destination D:\
```

---

## task 3: debugging tools & PicoProbe hardware debugging (pair / 2 boards)

use hardware breakpoints, data watchpoints, memory inspection & register stepping via OpenOCD & GDB.

extra hardware:
- Pico (flashed with `picoprobe.uf2`)
- 5x jumper wires (3x SWD: GP2/SWCLK, GP3/SWDIO, GND; 2x UART: GP4/TX, GP5/RX)

target & launch selection in VSCode:
- configure CMake extension: right click target binary & select "Set as Build Target", then "Set as Launch/Debug Target".

![Set Build & Launch Target](settarget.PNG)

- status bar indicates selected target with hammer (build) & rocket (launch) icons.

![Target Selection Confirmation](selectdebug.png)

- launch debugging session via Run & Debug panel (`F5`). VSCode flashes target via OpenOCD SWD interface & halts at `main()`.

![Active Hardware Debugging Session](debugging.png)

GDB debugging commands:
```text
# set line breakpoint
(gdb) break main
# set data watchpoint on memory write
(gdb) watch -l *(uint32_t *)0x20001000
# inspect call stack frame & registers
(gdb) backtrace
(gdb) info registers
# step over & step into instructions
(gdb) next
(gdb) step
# resume execution
(gdb) continue
```

---

## task 4: optimization exercises

implement fast equivalents of 4 computationally expensive functions on RP2040 ARMv6-M architecture.

refer to [optimise/README.md](optimise/README.md) for benchmark specifications, profiling rules & disassembly inspection instructions.

---

## task 5: PID controller debugging (`pid.c`)

correct at least 5 syntax errors & 5 logical errors in `pid.c` against the control algorithm pseudocode.

control algorithm pseudocode:
```text
// initialize parameters
Kp = 1.0, Ki = 0.1, Kd = 0.01
setpoint = 100.0, current_value = 0.0, integral = 0.0, prev_error = 0.0
time_step = 0.1, num_iterations = 100

// main simulation loop
for i = 0 to num_iterations - 1:
    error = setpoint - current_value
    integral += error
    derivative = error - prev_error
    control_signal = Kp * error + Ki * integral + Kd * derivative
    prev_error = error
    motor_response = control_signal * 0.1
    current_value += motor_response
    print "Iteration ", i, ": Control Signal = ", control_signal, ", Current Position = ", current_value
    sleep for time_step
```

defects to fix in `pid.c`:
- syntax error 1 (line 32): missing semicolon `;` after variable declaration.
- syntax error 2 (line 35-47): missing closing curly brace `}` for `for` loop body.
- syntax error 3 (line 36): passing `prev_error` as float value instead of pointer required by `compute_pid` signature.
- syntax error 4 (line 44): implicit declaration of `usleep` without `<unistd.h>` header inclusion (or missing Pico SDK `sleep_ms` call).
- syntax error 5 (line 42): comparison operator `==` used instead of assignment `=`.
- logical error 1 (lines 5-7): gain constants do not match the pseudocode specification (`Kp = 1.0`, `Ki = 0.1`, `Kd = 0.01`).
- logical error 2 (line 12): error sign is inverted.
- logical error 3 (line 18): extraneous derivative scaling factor.
- logical error 4 (line 20): `prev_error` updated with `current_value` instead of `error`.
- logical error 5 (line 38): motor response scaling factor mismatch against pseudocode.
- logical error 6 (line 40): format specifier mismatch in `printf` passing floating-point variable to integer `%d`.
- logical error 7 (line 44): time step delay conversion truncates fractional seconds to 0 microseconds.

host compilation & verification:
```sh
# compile corrected pid.c on host
gcc -Wall -Wextra -o pid pid.c -lm
# execute simulation
./pid
```

---

## task 6: Bug Hunt #6 (pair / 2 boards)

extra hardware:
- Pico (flashed with `picoprobe.uf2`)
- 3x jumper wires

refer to [bughunt readme](bughunt/README.md) for more.
