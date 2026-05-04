# ProtoDuck

![Primary MCU](https://img.shields.io/badge/Primary_MCU-RP2350B-0052cc?style=for-the-badge)
![Wireless](https://img.shields.io/badge/Wireless-ESP32--C6-e63946?style=for-the-badge)
![EDA](https://img.shields.io/badge/Designed_in-EasyEDA-2a9d8f?style=for-the-badge)

A powerful abomination of **RP2350B** and **ESP32-C6** chips that allows more GPIO pins and wireless connectivity with more processing power. The board is packed with heavy-duty protection features like static shock absorbers, eFuses, and stable voltage regulators designed specifically to prevent shorting and frying the board. 

If you've ever burned out a microcontroller by accident, this board was built for you.

## Board Visuals

**3D Visualisation**
<img width="353" height="568" alt="Screenshot 2026-05-04 at 10 08 20 PM" src="https://github.com/user-attachments/assets/f22d0bc2-f45c-41ee-8e2e-c8e2e5af5423" />

**Schematic Drawing**
<img width="4034" height="2836" alt="Schematic_ProtoDuck_v1_2026-05-04" src="https://github.com/user-attachments/assets/43f26e2d-de9b-4a79-9e96-623b85040569" />

## System Architecture

The ProtoDuck runs on a dual-microcontroller setup to prevent bottlenecking:

* **Primary MCU (RP2350B):** The main brain. Handles PC communication, GPIO pins, and runs user code. One ARM core is dedicated purely to system tasks and ESP communication, while the remaining cores are fully available for user code.
* **Wireless Co-Processor (ESP32-C6):** Connected via RX/TX. Exclusively handles WiFi (2.4GHz), Bluetooth 5+, and Zigbee. A custom HAL is being developed so users can access wireless features without eating into their own processing overhead.

## Power & Protection

The power circuit is designed to take a beating so the MCUs don't have to. It features a `CH224K` chip for proper USB Type-C power negotiation, an `SRV05-4-P-T7` TVS Diode to absorb static shocks on data lines, and a `TPS54302DDCR_C311983` Step-Down Converter to cleanly drop the negotiated voltage to a stable 3.3V.

> [!NOTE]  
> **Hardware eFuse (`TPS259271DRCR`)**
> The board features an eFuse that automatically trips upon overloading or overdrawing. When tripped, a bright LED illuminates and power is cut off from the rest of the board. To clear the fault, you must physically unplug and replug the cable.

## Three-Chip Storage System

To maximize security and prevent corruption, memory is isolated into three separate 128Mbit Flash chips (`GD25Q128ESIGR`), alongside the ESP32-C6's internal flash:

1. **`USR_FC` (User Chip):** Stores user code. Lowest priority at boot. Only accessible via PC/GUI for updates.
2. **`RP_FC` (Firmware Chip):** Stores the bootloader, kernel, and HALs. Connected via QSPI to the RP2350B. Uses an A/B partition system (similar to Android OTA updates) for safe flashing.
3. **`BC_FC` (Backup Chip):** Contains compressed firmware and an emergency bootloader for both MCUs in case of total system corruption.

## Pinout & Peripherals

The board maximizes usable IO and includes onboard peripherals (Reset Button, User Flash Button, System Status LED, Buzzer) for quick testing.

| Type | Count | Details |
| :--- | :--- | :--- |
| **Digital** | 21 | Programmable for PWM, I2C, SPI, etc. |
| **Analog** | 8 | Standard ADC inputs |
| **VBUS Protected** | 3 | Safely routed from USB |
| **3.3V Power** | 5 | Regulated output |
| **GND** | 9 | System ground |
| **I2C Header** | 1 | Dedicated plug-and-play header for displays |
| **Debug Header** | 1 | SWCLK & SWDIO |

## Emergency Recovery

If the software fails completely, ProtoDuck has physical hardware pins dedicated to recovering the board. Shorting these 1x1 male pin headers to `GND` triggers the following:

> [!IMPORTANT]  
> **`SYS-R` (System Restore)**
> Automatically fetches the compressed firmware from the `BC_FC` backup chip and safely reflashes the primary `RP_FC` chip.

> [!CAUTION]  
> **`FW-FLS` (Firmware Flash)**
> The nuclear option. If `SYS-R` fails, shorting this pin forces the RP flash chip to mount as a mass storage device on your PC. You can then drag-and-drop the firmware files to manually recover.

## Project Files & Links

This project was built using EasyEDA. The `.json` project files are included in this repository. 

* [View the live project on EasyEDA](https://u.easyeda.com/join?type=project&key=3605d2a7cc6ffb6cb2530e1b440ef092&inviter=137832c330b146bcba1d0ab581e94424)

---
*API Documentation & Firmware are currently a work in progress.*
