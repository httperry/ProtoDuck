# ProtoDuck

![Primary MCU](https://img.shields.io/badge/Primary_MCU-RP2350B-0052cc?style=for-the-badge)
![Wireless](https://img.shields.io/badge/Wireless-ESP32--C6-e63946?style=for-the-badge)
![EDA](https://img.shields.io/badge/Designed_in-EasyEDA-2a9d8f?style=for-the-badge)

ProtoDuck is a general use microcontroller designed by combining two separate chips (`RP2350B` & `ESP32-C6`) to provide the maximum potential to the microcontroller. The board was designed to solve these problems faced by other microcontrollers in the market:
1. Little to no circuit protection (Try accidentally touching a few pins using jumper wires0
2. Less GPIO pins (For ___some___ use cases)
3. Less wireless connectivity
4. Lack of libraries in software

The aim of ProtoDuck is to provide a safe baseline for tinkerers to be able to safely continue their journey in hardware

Please note that this is the first PCB I ever designed and may contain many flaws, you are free to point them out via the issues section and if possible I will try fixing them!

## Board Visuals

**3D Visualisation Front**

<img width="353" height="568" alt="Screenshot 2026-05-04 at 10 08 20 PM" src="https://github.com/user-attachments/assets/f22d0bc2-f45c-41ee-8e2e-c8e2e5af5423" />

**3D Visualisation Back**

<img width="432" height="699" alt="Screenshot 2026-05-17 at 11 40 32 AM" src="https://github.com/user-attachments/assets/04a11c12-674c-4cc4-9ba9-1989beb00fc1" />

**Schematic Drawing**

<img width="4034" height="2836" alt="Schematic_ProtoDuck_v1_2026-05-04" src="https://github.com/user-attachments/assets/43f26e2d-de9b-4a79-9e96-623b85040569" />

**Top Layer PCB**

<img width="429" height="697" alt="Screenshot 2026-05-17 at 11 20 19 AM" src="https://github.com/user-attachments/assets/eb2c0b60-8560-4d63-b409-3fda4f3a40f6" />

**Bottom Layer PCB**

<img width="421" height="696" alt="Screenshot 2026-05-17 at 11 20 46 AM" src="https://github.com/user-attachments/assets/ce03860e-2d92-443c-9bfc-c9c33bb1100c" />

**Middle Layer PCB**

<img width="425" height="695" alt="Screenshot 2026-05-17 at 11 21 07 AM" src="https://github.com/user-attachments/assets/78e6bfae-61c3-4a00-bd25-2693626e1690" />

## System Architechture

The ProtoDuck runs on a dual-microcontroller setup to prevent bottlenecking:

* **Primary Chip (RP2350B):** The main brain. Handles PC communication, GPIO pins, and runs user code. One ARM core is dedicated purely to system tasks and ESP communication, while the remaining cores are fully available for user code.
* **Wireless Co-Processor (ESP32-C6):** Connected via RX/TX. Exclusively handles WiFi (2.4GHz), Bluetooth 5+, and Zigbee. A custom HAL is being developed so users can access wireless features without eating into their own processing overhead.

## Power & Protection

The power circuit is designed while keeping the MCUs in mind to prevent any load on them. It features a `CH224K` chip for proper USB Type-C power negotiation, an `SRV05-4-P-T7` TVS Diode to absorb static shocks on data lines, and a `TPS54302DDCR_C311983` Step-Down Converter to cleanly drop the negotiated voltage to a stable 3.3V.

> [!NOTE]  
> **Hardware eFuse (`TPS259271DRCR`)**
> The board features an eFuse that automatically trips upon overloading or overdrawing. When tripped, a bright LED illuminates and power is cut off from the rest of the board. To clear the fault, you must physically unplug and replug the cable.

## Data Storage

To maximize security and prevent corruption, memory is isolated into three separate 128Mbit Flash chips (`GD25Q128ESIGR`), alongside the ESP32-C6's internal flash:

1. **`USR_FC` (User Chip):** Stores user code. Lowest priority at boot. Only accessible via PC/GUI for updates.
2. **`RP_FC` (Firmware Chip):** Stores the bootloader, kernel, and HALs. Connected via QSPI to the RP2350B. Uses an A/B partition system (similar to Android OTA updates) for safe flashing.
3. **`BC_FC` (Backup Chip):** Contains compressed firmware and an emergency bootloader for both MCUs in case of total system corruption.

There are physical pins on board that allow the user to flash the chips in case of recovery

## Pinout & Peripherals

The board maximizes usable IO and includes onboard peripherals (Reset Button, User Flash Button, System Status LED, Buzzer) for quick testing.
| Type               | Count | Pins             | Details                                     |
|--------------------|-------|------------------|---------------------------------------------|
| **Digital**        | 21    | `D0`-`D20`       | Programmable for PWM, I2C, SPI, etc.        |
| **Analog**         | 8     | `A0`-`A7`        | Standard ADC inputs                         |
| **VBUS Protected** | 3     | `VBUS_PROT`      | Safely routed from USB                      |
| **3.3V Power**     | 5     | `+3.3V`          | Regulated output                            |
| **GND**            | 9     | `GND`            | System ground                               |
| **I2C Header**     | 1     | `I2C`            | Dedicated plug-and-play header for displays |
| **Debug Header**   | 1     | Labeled on Board | SWCLK & SWDIO                               |

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

## Bill of Materials (BOM)
[BOM](https://raw.githubusercontent.com/httperry/ProtoDuck/refs/heads/main/BOM.csv)
