# STM32 FreeRTOS UART Communication (Producer-Consumer Model)

This project establishes an asynchronous serial communication (UART) and Real-Time Operating System (FreeRTOS) based "Producer-Consumer" infrastructure between two STM32 microcontrollers with different architectures (STM32F103RBT6 and STM32G030C8T6).

## 🛠️ Hardware Used
* **Producer (Transmitter):** STM32 Nucleo-F103RBT6
* **Consumer (Receiver):** Demedukit v1.1 (STM32G030C8T6)
* **Connection Protocol:** UART (115200 Baud Rate, 8 Bits, No Parity, 1 Stop Bit)

### Hardware Wiring Diagram
| Nucleo (Transmitter) | Demedukit (Receiver) | Description |
| :--- | :--- | :--- |
| GND | GND | Common ground reference (Critical) |
| PA9 (TX / D8 Pin) | S1 (RX Pin) | Data transmission line |

---

## ⚙️ Project Architecture and Stages

The main objective of this project is to process incoming data from the outside world at random intervals securely and without blocking the CPU (non-blocking) using an RTOS architecture.

1. **Producer Stage (Nucleo):** Operating without an OS, a standard `while(1)` loop generates and sends a string message in the format `COUNT: X\n` (where X is an incrementing counter) over UART every second.
2. **Postman Stage (Hardware Interrupt):** On the Demedukit side, the UART RX interrupt is enabled using `HAL_UART_Receive_IT`. Every received byte is caught inside the `HAL_UART_RxCpltCallback` function and instantly pushed to a FreeRTOS Queue named `DataQueue`.
3. **Consumer Stage (FreeRTOS Task):** A high-priority FreeRTOS task named `ReceiverTask` running on the Demedukit continuously listens to the queue (`osWaitForever`). The moment new data falls into the queue, the task wakes up, processes the data, and toggles the state of the red LED on the board.

---

## 🐛 Encountered Errors and Solutions

During the development process, hardware limitations and RTOS conflicts were encountered. The engineering solutions to these issues are detailed below:

### 1. FreeRTOS SysTick Conflict (Timebase Source Warning)
* **Issue:** In STM32 projects, delay functions (`HAL_Delay`) use the `SysTick` timer by default. However, FreeRTOS also requires the same `SysTick` source for its own core timing (Heartbeat/Tick). Both systems trying to use the same source leads to a system freeze/hang.
* **Solution:** In the STM32CubeMX (IOC) interface, navigating to the `System Core -> SYS` menu, the **Timebase Source** setting was shifted from `SysTick` to an available hardware timer (**TIM1**).

### 2. Insufficient Memory (RAM Overflowed) and Linker Settings
* **Issue:** The STM32G030C8T6 microcontroller on the Demedukit has only 8 KB of RAM. During compilation, a `region RAM overflowed by 584 bytes` error occurred. This was because the default System Heap reserved by STM32CubeIDE and the memory allocated by FreeRTOS overlapped, causing memory exhaustion.
* **Solution Steps:**
  1. Initially, the memory allocated in the FreeRTOS config file (`TOTAL_HEAP_SIZE`) was optimized and reduced to 1500 bytes.
  2. **The Definitive Solution:** In the STM32CubeMX interface, navigated to the **Project Manager** tab at the top. Selected **Project** from the left menu and scrolled down to the **Linker Settings** section. Since FreeRTOS manages its own Heap (`heap_4.c`), the standard dead memory allocated for the system was unnecessary. Therefore, the **Minimum Heap Size** value was changed from `0x200` to **`0x0`**. The **Minimum Stack Size** reserved for interrupts was left at **`0x200`** (512 Bytes). This intervention recovered the idle RAM, and the project compiled successfully with zero errors.

### 3. LED Toggling at an Invisible Speed
* **Issue:** Even though the code was successfully flashed and the RTOS queue was working, there was no visible reaction from the LED.
* **Solution:** When the `COUNT: X\n` text is sent at a 115200 Baud Rate, all bytes in the packet reach the receiver in a fraction of a millisecond. Because the processor matched this speed and toggled the LED multiple times for each character, the human eye couldn't perceive it, making the LED appear constantly off (or very dim). An **`osDelay(50);`** (50 milliseconds RTOS delay) was added right below the `HAL_GPIO_TogglePin` command inside the `ReceiverTask` to slow down the hardware response to a frequency visible to the eye.

---
*Development: [yunus-kunduz](https://github.com/yunus-kunduz)*