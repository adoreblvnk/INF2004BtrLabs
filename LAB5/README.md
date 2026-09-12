# LAB 5: FreeRTOS & Real-Time Concepts

multitasking, preemptive scheduling, inter-task communication & synchronisation on Raspberry Pi Pico W using FreeRTOS.

---

## section 1: FreeRTOS setup & configuration

configure FreeRTOS-Kernel & build environment for Raspberry Pi Pico W.

extra hardware:
- mobile phone with WiFi hotspot enabled

![FreeRTOS Kernel Folder](img/freertosfolder.png)
- FreeRTOS-Kernel source repository contains the core real-time operating system scheduler, portable layers & heap allocators.

CMake environment configuration:
- configure three key CMake environment variables for Pico W wireless & FreeRTOS compilation:
  - `FREERTOS_KERNEL_PATH`: path to extracted FreeRTOS-Kernel directory.
  - `WIFI_SSID`: wireless access point SSID (e.g. mobile hotspot).
  - `WIFI_PASSWORD`: wireless access point password.

![CMake Tools](img/CMakeTools.png)
![CMake Environment](img/CmakeEnvironment.png)
- CMake Tools extension environment configuration in VS Code.

USB serial output configuration:
- add `pico_enable_stdio_usb(picow_freertos_ping_sys 1)` in `CMakeLists.txt` to route standard output via USB CDC serial.

disabling SMP (Symmetric Multiprocessing):
- modify `FreeRTOSConfig.h` to pin execution strictly to a single core (`configNUM_CORES 1`).

```c
#if FREE_RTOS_KERNEL_SMP // set by the RP2040 SMP port of FreeRTOS
/* SMP port only */
#define configNUM_CORES                         1
#define configTICK_CORE                         0
#define configRUN_MULTIPLE_PRIORITIES           1
#define configUSE_CORE_AFFINITY                 0
#endif
```

![Disable SMP](img/NoSMP.PNG)
- FreeRTOSConfig.h configuration disabling multi-core SMP scheduling.
- RP2040 contains two identical ARM Cortex-M0+ cores. Enabling SMP allows two tasks to execute simultaneously across both cores, creating true hardware concurrency & multi-core race conditions. Disabling SMP restricts scheduling to core 0, ensuring deterministic priority-based preemption.

![CMake Ping Target](img/CMakePing.png)
- CMake Tools project outline displaying configured `picow_freertos_ping` build targets.

![Initial Ping Output](img/task1.png)
- serial console output verifying WiFi network association & ICMP ping transmission under FreeRTOS.

build & flash `picow_freertos_ping`:

macOS:
```sh
# clone FreeRTOS-Kernel if not already cloned
mkdir -p ~/pico
git clone https://github.com/FreeRTOS/FreeRTOS-Kernel.git ~/pico/FreeRTOS-Kernel
# copy sdk import helper
cp ~/pico/pico-sdk/external/pico_sdk_import.cmake .
# configure & build
mkdir -p build && cd build
cmake -DPICO_BOARD=pico_w -DFREERTOS_KERNEL_PATH=$HOME/pico/FreeRTOS-Kernel ..
make -j8 picow_freertos_ping_sys
# flash to pico (bootsel mode)
cp picow_freertos_ping_sys.uf2 /Volumes/RPI-RP2
# open serial monitor (Ctrl-A Ctrl-\ to exit)
screen /dev/tty.usbmodem* 115200
```

windows (powershell):
```powershell
# clone FreeRTOS-Kernel
New-Item -ItemType Directory -Force -Path C:\pico
git clone https://github.com/FreeRTOS/FreeRTOS-Kernel.git C:\FreeRTOS-Kernel-main
# copy sdk import helper
Copy-Item C:\pico\pico-sdk\external\pico_sdk_import.cmake .
# configure & build
New-Item -ItemType Directory -Force -Path build
cd build
cmake -DPICO_SDK_PATH="C:\pico\pico-sdk" -DPICO_BOARD=pico_w -DFREERTOS_KERNEL_PATH="C:\FreeRTOS-Kernel-main" ..
cmake --build . --target picow_freertos_ping_sys
# flash to pico
Copy-Item picow_freertos_ping_sys.uf2 -Destination D:\
```

---

## task 1: Blinking LED task

create concurrent LED blinking task alongside network operations.

![Blinking LED Task](img/task2.png)
- serial monitor output showing concurrent execution of LED status messages alongside network ping logs.

task function architecture:
- tasks execute as infinite loops (`while(true)`) & must never return or exit their entry function.

```c
#include <stdio.h>
#include "hardware/gpio.h"

void led_task(__unused void *params) {
    while (true) {
        vTaskDelay(pdMS_TO_TICKS(2000));
        cyw43_arch_gpio_put(CYW43_WL_GPIO_LED_PIN, 1);
        vTaskDelay(pdMS_TO_TICKS(2000));
        cyw43_arch_gpio_put(CYW43_WL_GPIO_LED_PIN, 0);
    }
}
```

task instantiation:
- instantiate task via `xTaskCreate()` inside `vLaunch()` before starting scheduler:

```c
TaskHandle_t ledtask;
xTaskCreate(led_task, "TestLedThread", configMINIMAL_STACK_SIZE, NULL, 5, &ledtask);
```

task state lifecycle:

```
                       task is chosen
                       by the scheduler
          +---------+ ---------------> +---------+
          |  READY  |                  | RUNNING |
          +---------+ <--------------- +---------+
             ^   ^      preempted by a    |    |
             |   |      higher-priority   |    |
             |   |      task, or          |    |
             |   |      time-sliced       |    |
             |   |                        |    |
             |   |  event occurs, or      |    | vTaskDelay(),
             |   |  timeout expires       |    | xQueueReceive(),
             |   +------------------------+    | xSemaphoreTake(), ...
             |            +---------+          |
             |            | BLOCKED | <--------+
             |            +---------+
             |  vTaskResume()              vTaskSuspend()
             |            +-----------+
             +----------- | SUSPENDED | <-------------
                          +-----------+
```

key task states:
- Running: task is actively executing instructions on the CPU core. On single-core RP2040, exactly one task is Running at any instant.
- Ready: task is eligible for execution but waiting because an equal or higher priority task is currently Running.
- Blocked: task is waiting for a temporal delay, queue message, or synchronization event. Blocked tasks consume zero CPU execution time.
- Suspended: task is explicitly removed from scheduler execution via `vTaskSuspend()` until awakened by `vTaskResume()`.

Blocked state vs busy-waiting:
- `sleep_ms(2000)`: busy-waits by burning CPU cycles in Ready/Running state, starving equal & lower priority tasks.
- `vTaskDelay(pdMS_TO_TICKS(2000))`: transitions caller into Blocked state, allowing other tasks to execute during delay.
- Idle task: executes at priority 0 (`tskIDLE_PRIORITY`) whenever all application tasks are Blocked.

priority-based preemptive scheduling:
- task priority ranges from `0` (`tskIDLE_PRIORITY`) up to `configMAX_PRIORITIES - 1`. Larger integer numbers designate higher priority.
- scheduling rule: the highest-priority Ready task Runs immediately.
- preemption: a higher-priority task becoming Ready preempts running lower-priority tasks instantly. Tasks of identical priority are time-sliced round-robin on each tick interrupt (`configUSE_TIME_SLICING`).

`pdMS_TO_TICKS()` macro:
- `vTaskDelay()` expects duration in raw scheduler ticks, not milliseconds.
- `pdMS_TO_TICKS(2000)` converts milliseconds to ticks at compile time using `configTICK_RATE_HZ`. Direct tick values break timing if kernel tick rate changes.

---

## task 2: Periodic temperature sensor task

sample RP2040 internal temperature sensor at regular intervals without cumulative timing drift.

![Periodic Temperature Task](img/task3.png)
- serial console displaying periodic temperature telemetry sampled from RP2040 ADC channel 4.

ADC hardware setup:
- RP2040 internal temperature sensor connects to ADC channel 4.
- include `hardware/adc.h` in C source & append `hardware_adc` to `target_link_libraries` in `CMakeLists.txt`.

```c
float read_onboard_temperature(void) {
    /* 12-bit conversion, assume max value == ADC_VREF == 3.3 V */
    const float conversionFactor = 3.3f / (1 << 12);

    float adc = (float)adc_read() * conversionFactor;
    float tempC = 27.0f - (adc - 0.706f) / 0.001721f;

    return tempC;
}
```

eliminating drift with `xTaskDelayUntil()`:
- `vTaskDelay()` measures delay relative to invocation time, adding loop execution overhead to each period & causing cumulative drift.
- `xTaskDelayUntil()` computes delay relative to an absolute target tick count (`xLastWakeTime`), guaranteeing a fixed execution cadence.

```c
void temp_task(__unused void *params) {
    float temperature = 0.0;
    TickType_t xLastWakeTime;
    const TickType_t xPeriod = pdMS_TO_TICKS(1000);

    adc_init();
    adc_set_temp_sensor_enabled(true);
    adc_select_input(4);

    xLastWakeTime = xTaskGetTickCount(); // initialise once before loop

    while (true) {
        xTaskDelayUntil(&xLastWakeTime, xPeriod); // wake at fixed cadence
        temperature = read_onboard_temperature();
        printf("Onboard temperature = %.02f C\n", temperature);
    }
}
```

task creation:
```c
TaskHandle_t temptask;
xTaskCreate(temp_task, "TestTempThread", configMINIMAL_STACK_SIZE, NULL, 8, &temptask);
```

drift vs jitter:

| Metric | Definition | Root cause | Mitigation |
|---|---|---|---|
| **Drift** | Cumulative error in average period over time | Delaying relative to loop completion (`vTaskDelay`) | `xTaskDelayUntil()` tracking absolute ticks |
| **Jitter** | Cycle-to-cycle variation around scheduled wake time | Preemption by higher-priority tasks, ISR duration | Priority tuning & bounding critical sections |

---

## task 3: Inter-task communication & filtering

decouple sampling cadence from signal processing using FreeRTOS communication primitives.

![Moving Average Task](img/task4.png)
- serial monitor output displaying raw temperature readings alongside computed 4-sample moving averages.

architectural separation:
- sampling task operates on strict timing deadlines; filtering task runs on soft deadlines. Decoupling both via kernel buffers ensures heavy processing never delays sensor acquisition.

Message Buffer vs Queue vs Stream Buffer:

| Primitive | Item size | Writers | Readers | Primary use case |
|---|---|---|---|---|
| **Queue** | Fixed (set at creation) | Multiple | Multiple | Default thread-safe IPC for discrete messages |
| **Message buffer** | Variable (length-prefixed) | **Single (1)** | **Single (1)** | Dedicated single-producer single-consumer byte packets |
| **Stream buffer** | Continuous byte stream | **Single (1)** | **Single (1)** | Continuous unstructured byte streams (UART, ADC DMA) |

Message Buffer constraints:
- Message Buffers are strictly single-producer & single-consumer (SPSC). Violating single-reader or single-writer contracts causes memory corruption.
- data is copied by value, ensuring memory safety without shared pointer lifetime hazards.

Message Buffer creation & sending:
```c
#include "message_buffer.h"

#define mbaTASK_MESSAGE_BUFFER_SIZE (60)
static MessageBufferHandle_t xControlMessageBuffer;

// create in vLaunch before starting scheduler
xControlMessageBuffer = xMessageBufferCreate(mbaTASK_MESSAGE_BUFFER_SIZE);

// send from temp_task
xMessageBufferSend(xControlMessageBuffer, (void *)&temperature, sizeof(temperature), 0);
```

moving average task implementation:
```c
void avg_task(__unused void *params) {
    float fReceivedData;
    size_t xReceivedBytes;
    static float data[4] = {0};
    static int index = 0;
    static int count = 0;
    float sum = 0;

    while (true) {
        xReceivedBytes = xMessageBufferReceive(
            xControlMessageBuffer,
            (void *)&fReceivedData,
            sizeof(fReceivedData),
            portMAX_DELAY); // block until sample arrives

        sum -= data[index];
        data[index] = fReceivedData;
        sum += data[index];
        index = (index + 1) % 4;

        if (count < 4) count++;

        printf("Average Temperature = %0.2f C\n", sum / count);
    }
}
```

task creation:
```c
TaskHandle_t avgtask;
xTaskCreate(avg_task, "TestAvgThread", configMINIMAL_STACK_SIZE, NULL, 5, &avgtask);
```

reentrancy & `static` variable warning:
- `static` variables (`data`, `index`, `count`) allocate single memory instances shared across all invocations. If multiple tasks execute `avg_task`, they corrupt shared filter state. Reentrant tasks require private state structures passed via parameters.

---

## task 4: Mutual exclusion & priority inversion

safeguard shared unbuffered hardware resources & manage priority preemption anomalies.

shared resource race condition:
- `printf` accesses shared stdio buffers & USB serial hardware. Preempting a task mid-`printf` produces interleaved character corruption across serial logs.

critical sections:
- `taskENTER_CRITICAL()` & `taskEXIT_CRITICAL()` disable CPU interrupts globally.
- halts scheduler, timers & all ISRs. Critical sections must be limited to brief multi-instruction operations & never used around slow I/O or `printf`.

Mutex (`semphr.h`):
- mutual exclusion object with ownership tracking & priority inheritance.
- calling `xSemaphoreTake()` blocks contending tasks without spinning, relinquishing CPU to ready tasks.

```c
#include "semphr.h"

SemaphoreHandle_t xPrintMutex;

// create before scheduler start
xPrintMutex = xSemaphoreCreateMutex();

// protect shared printf
if (xSemaphoreTake(xPrintMutex, portMAX_DELAY) == pdTRUE) {
    printf("Onboard temperature = %.02f C\n", temperature);
    xSemaphoreGive(xPrintMutex);
}
```

synchronization primitives comparison:

| Primitive | Creation API | Ownership | Initial count | Primary purpose |
|---|---|---|---|---|
| **Mutex** | `xSemaphoreCreateMutex()` | Yes (locked to task) | 1 (unlocked) | Resource mutual exclusion with priority inheritance |
| **Binary semaphore** | `xSemaphoreCreateBinary()` | No owner | 0 (empty) | Unidirectional signaling between ISR & task |
| **Counting semaphore** | `xSemaphoreCreateCounting()` | No owner | Configurable | Resource pool tracking or event counting |

priority inversion & priority inheritance:
- priority inversion occurs when low-priority task (L) holds a mutex required by high-priority task (H), & medium-priority task (M) preempts L, causing H to wait indefinitely behind unrelated task M.
- priority inheritance temporarily elevates L's priority to match H until L releases the mutex, preventing M from preempting L & bounding inversion duration.

gatekeeper task alternative:
- route all formatting requests to a dedicated print task via a FreeRTOS Queue (`xQueueSend`), eliminating mutex contention & priority inversion hazards entirely.

deferred interrupt handling:
- ISR handles time-critical hardware clearing & signals handler task using `xSemaphoreGiveFromISR()` or `xQueueSendFromISR()`.
- invoke `portYIELD_FROM_ISR(xHigherPriorityTaskWoken)` on ISR exit to trigger immediate preemption if higher-priority task became Ready.

---

## task 5: Bug Hunt #5

refer to bughunt readme for more.
