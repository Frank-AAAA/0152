# 0152
# Embedded Systems Coursework – FreeRTOS Snake Game (STM32F401 + SSD1306 I2C + ADC DMA Joystick + SystemView)

This submission contains an STM32CubeIDE project implementing a Snake game using FreeRTOS (CMSIS-RTOS v2), an SSD1306 128×64 OLED over I2C, and a 2-axis joystick via ADC1 Scan + DMA. Runtime behaviour is profiled using SEGGER SystemView over UART.

Project folder: `0152/`

---

## 1. What is implemented (Features)

### Core gameplay
- Snake movement using joystick direction input (4-way)
- Food spawning and score/length increase
- Collision detection (wall / self) → GAME OVER
- START screen (WAIT mode) → joystick movement starts the game
- PAUSE/RESUME during gameplay via joystick SW button
- Restart from GAME OVER via SW button

### Real-time / RTOS design (coursework-relevant)
- Deterministic periodic logic using `osDelayUntil` where applicable
- Input acquisition separated from game update and rendering
- Display updates decoupled using event/signal style synchronization to prevent I2C stalls from blocking gameplay
- SystemView runtime profiling hooks to evidence task scheduling and CPU utilisation

---

## 2. Hardware setup (Wiring)

### 2.1 MCU
- Target MCU family: **STM32F401xE**

### 2.2 OLED: SSD1306 (I2C1)
Connections:
- VCC → 3.3V
- GND → GND
- SCL → **PB8 (I2C1_SCL)**
- SDA → **PB9 (I2C1_SDA)**

I2C address used by the driver:
- **7-bit: 0x3C**
- **8-bit: 0x78** (used in HAL transmit calls)

### 2.3 Joystick (ADC + SW)
ADC axes (ADC1, scan + DMA, 2-channel buffer):
- X axis → **PA0 / ADC1_IN0**
- Y axis → **PA1 / ADC1_IN1**

Button (active-low):
- SW → **PA8** (GPIO input with pull-up)

### 2.4 UART for SystemView
- USART: **USART2**
- TX: **PA2 (USART2_TX)**
- RX: **PA3 (USART2_RX)**
- Baud rate used by SystemView init: **921600**

---

## 3. Software environment
- IDE: **STM32CubeIDE**
- HAL: STM32 HAL drivers
- RTOS: **FreeRTOS** (via **CMSIS-RTOS v2**)
- OLED driver: Minimal framebuffer SSD1306 driver (`Core/Src/ssd1306_min.c`)
- Profiling: SEGGER **SystemView** (under `Middlewares/Third_Party/SEGGER/`)

---

## 4. Build & run instructions (Reproducibility)

### 4.1 Import project
1. Open **STM32CubeIDE**
2. `File → Import → Existing Projects into Workspace`
3. Select the directory that contains `0152/`
4. Ensure the project is detected and imported

### 4.2 Build
- `Project → Build Project`

### 4.3 Flash and execute
- Connect the board via USB (ST-LINK)
- `Run → Debug` (or `Run`)

Key runtime initialisation:
- ADC DMA starts with a 2-sample buffer (X/Y):
  - `HAL_ADC_Start_DMA(&hadc1, (uint32_t*)adc_buf, 2);`
- Scheduler starts via `osKernelStart()`

---

## 5. How to play (User instructions)

### 5.1 State flow
- **START / WAIT**: OLED shows “START”, snake does not move
- **PLAY**: snake moves, eats food, score increases
- **PAUSE**: gameplay frozen, display remains responsive
- **GAME OVER**: death screen after collision

### 5.2 Controls
- Move: joystick direction (ADC X/Y)
- Start: move joystick once on START screen
- Pause/Resume: press **SW (PA8)** during PLAY
- Restart after death: press **SW (PA8)** on GAME OVER

---

## 6. RTOS architecture (for marking)

### 6.1 Task decomposition (responsibility-driven)
The application separates concerns into dedicated tasks to improve determinism and avoid peripheral blocking:

| Task / Module | Responsibility | Timing / Notes |
|---|---|---|
| Input task | Reads ADC DMA buffer and converts analog values to direction events; applies thresholds/dead-zone | Designed to be lightweight and periodic |
| Game task | Main game tick: movement, collisions, food/score, mode transitions | Uses deterministic tick scheduling (`osDelayUntil`) where applicable |
| Display control / worker tasks | Renders to framebuffer and performs I2C transfer to OLED | Triggered via events; throttles updates to prevent I2C from dominating CPU time |

### 6.2 Inter-task synchronisation
- CMSIS-RTOS primitives (e.g., **event flags**) are used to signal frame-ready / render-needed events.
- Display is event-driven so that I2C transfers do not stall gameplay logic.
- Shared state is kept minimal; tasks interact via explicit signals rather than continuous polling.

---

## 7. SystemView profiling (how to reproduce traces)

SystemView is initialised in `main.c` using:
- `SEGGER_SYSVIEW_Conf();`
- `SEGGER_SYSVIEW_Start();`
- `SEGGER_UART_init(921600);`
- `HIF_UART_EnableTXEInterrupt();`

### 7.1 Capturing a trace
1. Ensure USART2 wiring is correct (PA2/PA3) and baud rate is 921600
2. Open SEGGER SystemView on the host PC
3. Configure the target connection to match the UART settings
4. Start recording while the game runs (START → PLAY → PAUSE → GAME OVER)

### 7.2 What to look for (marking evidence)
- Task switching behaviour (Input/Game/Display)
- CPU utilisation and relative runtimes
- The absence of long “blocking gaps” in gameplay when OLED updates occur

If SystemView is not required for the demo, the init calls can be disabled (removing SEGGER sources from build) to simplify runtime.

---

## 8. Testing & debugging notes (coursework reflection)

### Basic functional tests performed
- START screen shown on boot; no movement until input
- Direction changes follow joystick movement (dead-zone prevents jitter)
- Food generation does not overlap snake; score increments on eat
- Collision → GAME OVER; SW restarts correctly
- PAUSE freezes movement and resumes deterministically

### Typical issues & checks
- OLED blank:
  - verify I2C address (0x3C 7-bit / 0x78 8-bit)
  - confirm SCL/SDA wiring and pull-ups
- Unstable joystick:
  - confirm ADC channels and thresholds/dead-zone configuration
- SystemView integration impacts runtime:
  - ensure correct UART setup and that instrumentation does not conflict with timing-critical paths

---

## 9. Repository / submission structure

### Key paths
- `0152/Core/Src/freertos.c` : task creation and RTOS application logic
- `0152/Core/Src/main.c`     : HAL init + SystemView init + scheduler start
- `0152/Core/Src/ssd1306_min.c` : OLED driver (framebuffer + I2C burst update)
- `0152/0152.ioc` : CubeMX configuration (pins, peripherals, middleware)

### Clean submission recommendation
For a grader-friendly zip, keep:
- `Core/`, `Drivers/`, `Middlewares/`, `.ioc`, `.project`, `.cproject`, linker scripts

Recommended to remove (build artefacts / noise):
- `Debug/` or `Release/`
- `Core/Inc/Backup/` and `Core/Src/Backup/`
- temporary logs unless explicitly requested

---

## 10. Credits
- STM32 HAL: STMicroelectronics
- FreeRTOS: FreeRTOS / Amazon (via CMSIS-RTOS v2 wrapper in CubeMX)
- SEGGER SystemView: SEGGER Microcontroller GmbH
- SSD1306: framebuffer-based minimal driver included in this project
