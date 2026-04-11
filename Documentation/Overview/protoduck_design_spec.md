# ProtoDuck — Complete Design Specification

> **Status:** All major decisions locked ✅  
> **Last Updated:** April 5, 2026

---

## Hardware Overview

ProtoDuck is a dual-chip microcontroller designed for makers, tinkerers, and educators,
featuring robust protection, wireless connectivity, and a Python-first developer experience.

### Chips

| Chip | Role |
|---|---|
| **RP2350B** | Major chip — GPIO, USB, PC communication, user code runtime |
| **ESP32-C6-WROOM-1-N8** | Wireless chip — WiFi 6, BT 5.3, Zigbee, OTA downloader |

### Flash Storage (Three-Chip Architecture)

| Chip | Size | Format | Purpose |
|---|---|---|---|
| **USR_FC1** | 128Mbit (16MB) | FAT32 | 100% user space — `.py` files, `/lib/`, data |
| **RP_FC2** | 128Mbit (16MB) | A/B Partitioned | RP firmware, FreeRTOS, HAL, MicroPython |
| **BC_FC3** | 128Mbit (16MB) | FAT32 (2 partitions) | Factory backup only — compressed RP + ESP firmware |

> **No microSD card slot.** Users manage within the 16MB USR_FC1 budget.  
> **BC_FC3 is written ONCE during manufacturing. Never written to again.**

### RP_FC2 Layout

```
RP_FC2 (16MB)
├── Metadata Region (256KB)
│   ├── Active partition flag (A or B)
│   ├── Boot attempt counter (0–3, rollback trigger)
│   ├── confirmed flag (true after 3 clean boots)
│   ├── Firmware version strings
│   └── SHA256 hashes of both partitions
│
├── Partition A — Active (7.8MB)
│   ├── Stage 2 Bootloader      [~64KB]
│   ├── FreeRTOS Kernel          [~256KB]
│   ├── HAL + TrustZone setup    [~512KB]
│   └── MicroPython firmware     [~1.5MB]
│
└── Partition B — OTA Target (7.8MB)
    └── (identical structure, written via chunked OTA)
```

### BC_FC3 Layout

```
BC_FC3 (16MB, FAT32)
├── Partition 1: /rp_firmware.bin.gz   (compressed RP full firmware)
└── Partition 2: /esp_firmware.bin.gz  (compressed ESP full firmware)
```

---

## Power System

| Component | Role |
|---|---|
| **CH224K** | USB-PD negotiation — user selects 5V / 9V / 12V / 20V via PWR-SEL jumper header |
| **TPS259271DRCR** | eFuse — overcurrent protection, trips and cuts power, Red LED + Buzzer alert |
| **TPS54302DDCR** | Buck regulator — steps down to stable 3.3V for board operation |
| **SRV05-4-P-T7** | TVS Diode — ESD/static charge absorption |

**Fault behaviour:** eFuse trips → Red LED on + Buzzer beeps → power cut to chips → stays latched until USB unplug/replug (user acknowledges and fixes fault).

---

## Connections (RP2350B GPIO Map)

| GPIO | Pin Label | Connected To |
|---|---|---|
| 0 | RXD0 | ESP32-C6 (UART RX) |
| 1 | TXD0 | ESP32-C6 (UART TX) |
| 2 | DI | LED2 — RGB System Status LED |
| 3 | D20 | User GPIO |
| 4–7 | SPI | USR_FC1 (SO, CS#, SCLK, SI) |
| 8–11 | SPI | BC_FC3 (SO, CS#, SCLK, SI) |
| 12–13 | D0, D19 | User GPIO |
| 14 | PG | CH224K (power good) |
| 15–16, 18 | D18, D17, D16 | User GPIO |
| 17 | IO9 | ESP32-C6 control |
| 19 | USR_FLS | Tactile switch (3-sec hold = mount USR_FC1 as USB drive) |
| 20 | EN | ESP32-C6 enable |
| 21 | SYS_R | System Restore header |
| 22–36 | D1–D15 | User GPIO |
| 37–38 | SCL, SDA | CN1 — Display I2C header |
| 39 | — | Buzzer (via transistor) |
| 40–47 | A0–A7 | User Analog GPIO (ADC) |
| QSPI_SS | FW_FLS | Firmware flash pin header (RP_FC2 recovery) |
| QSPI_SS–SCLK | — | RP_FC2 (native QSPI boot interface) |

---

## Computing Architecture

### RP2350B Core Split

```
Core 0 — FreeRTOS [TrustZone SECURE world]
├── HAL (GPIO, SPI, UART, ADC drivers)
├── Flash manager (USR_FC1 r/w, BC_FC3 read, RP_FC2 OTA writer)
├── USB manager (CDC ↔ Mass Storage mode switching)
├── OTA engine (chunk receiver, hash verifier, metadata updater)
├── pch bridge server (deserialize calls → execute → serialize response)
└── ESP32 UART protocol handler (full-duplex async)

Core 1 — MicroPython [TrustZone NON-SECURE world]
├── Runs user code: boot.py (locked) → main.py (user entry point)
├── protoduck.* built-in namespace
├── pch / pchost module (PC bridge)
└── asyncio event loop
```

> **TrustZone enforcement:** ARM SAU/MPU memory-level. Non-Secure world (Core 1) cannot
> directly access flash drivers, UART to ESP, or RP_FC2. All hardware access goes through
> validated secure veneer function calls.

### ESP32-C6 Core Split

```
Core 0 — FreeRTOS (isolated)
├── Wireless stack (WiFi 6, BT 5.3, Zigbee/Thread 802.15.4)
├── HAL communicator (full-duplex UART binary protocol with RP2350B)
└── OTA downloader (HTTP range requests → chunk → UART → RP)

Core 1 — User wireless tasks (delegated from RP via protocol)
```

### ESP32-C6 Internal Flash Layout (8MB)

```
├── Bootloader              [64KB]
├── Partition Table         [4KB]
├── FreeRTOS App (OTA A)    [~3MB]  — current firmware
├── FreeRTOS App (OTA B)    [~3MB]  — ESP OTA target slot
├── NVS (encrypted)         [512KB] — WiFi credentials (AES-256 via ESP flash encryption)
└── OTA Data                [8KB]   — active slot tracking
```

---

## RP ↔ ESP Communication Protocol

- **Physical:** UART full-duplex async (GPIO 0/1 on RP, RXD0/TXD0 on ESP)
- **Control lines:** RP GPIO17 → ESP IO9, RP GPIO20 → ESP EN
- **Frame format:** Length-prefixed binary packets with command IDs
- **Direction:** Both chips can initiate — RP sends commands, ESP sends async events (WiFi connected, OTA chunk ready, etc.)

---

## User Python Environment

### MicroPython

- **Version:** MicroPython 1.23+ (RP2350 supported)
- **Location:** Bundled inside RP_FC2 A/B partitions (system-managed, not user-replaceable)
- **User entry point:** `main.py` on USR_FC1
- **System entry:** `boot.py` on USR_FC1 — **system-locked, not user-editable**
  - Initialises HAL session, pch bridge, mounts USR_FC1 filesystem
  - Adds `/lib` to `sys.path`

### File Layout on USR_FC1

```
USR_FC1 (16MB FAT32) — mounted as root /
├── boot.py      ← LOCKED — system-generated, not editable by user
├── main.py      ← User entry point
├── config.py    ← User config (optional)
├── mymodule.py  ← User modules
└── lib/         ← User libraries (installed via Library Manager or copied)
    └── mylib/
```

### ProtoDuck Module Namespace

All HAL features exposed via explicit imports:

```python
from protoduck import gpio
from protoduck import wireless
from protoduck import display
from protoduck import storage
from protoduck import system
```

User code cannot directly touch hardware — all calls go through TrustZone veneer functions.

---

## PC Host Bridge (`pch` / `pchost`)

### Purpose

Allows user code on the board to execute computationally heavy Python libraries
(numpy, pandas, matplotlib, etc.) on the connected PC's full CPython 3.12 instance.
Both `pch` and `pchost` are identical aliases to the same module.

### Import Syntax

```python
from pch import numpy as np          # explicit PC import
from pchost import pandas as pd      # identical, long-form alias
```

Regular MicroPython imports are unchanged — `pch`/`pchost` prefix is the explicit signal.

### Per-Call Timeout

```python
pch.timeout = 5.0  # global default (seconds)

# Per-call override using __timeout keyword
result = np.fft.fft(large_array, __timeout=30.0)
```

`__timeout` is intercepted by the pch proxy — never forwarded to the actual library.

### Exception Hierarchy

```
PCHError
├── PCHNotConnectedError    — USB not physically connected / app not running
├── PCHImportError          — library not found in PC venv
├── PCHTimeoutError         — no response within timeout window
├── PCHDisconnectedError    — USB dropped mid-computation
└── PCHSerializationError   — data couldn't be serialized for transfer
```

### Graceful Fallback Pattern

```python
try:
    from pch import numpy as np
except (PCHNotConnectedError, PCHImportError):
    import ulab.numpy as np  # on-board fallback
```

### PC App Python Runtime

- **Bundled CPython 3.12** shipped inside WinUI3 app install — no user Python install needed
- **Isolated venv** at `<AppInstall>/python/venv/` — zero conflict with user's system Python
- Library Manager UI = `pip install` into this venv
- `matplotlib` renders to WinUI3 app Visualization panel, not a desktop window

### Auto-Reconnection

- PC app always runs as a **Windows background service** (system tray)
- On USB disconnect: `PCHDisconnectedError` raised, HAL silently polls every 500ms
- On USB reconnect: session re-established automatically, `@pch.on_reconnect` callback fires
- No user action required

---

## Recovery System

### Path 1: SYS_R (Hardware or Software Triggered)

```
SYS_R pin triggered OR software command from app
    ▼
RP2350B bootloader mode
    1. Mount BC_FC3 over SPI
    2. Read /rp_firmware.bin.gz
    3. Decompress (streaming — no full RAM needed)
    4. Write to ACTIVE A/B partition
    5. Reboot
```

### Path 2: FW_FLS Header (Manual Brick Recovery)

```
Board enters USB Mass Storage mode — exposes RP_FC2 as "PROTODUCK_FW" drive
    ▼
User drops official .uf2 file
    ▼
Bootloader detects .uf2 → flashes to inactive partition → reboots
```

### USR_FLS Button (3-Second Hold)

Mounts USR_FC1 as USB Mass Storage drive ("PROTODUCK_USR") on connected PC.
User can drag-drop `.py` files, `/lib/` contents, etc. directly.
Single press / less than 3 seconds = ignored.

---

## OTA Update System

### Update Check
- ESP32-C6 background task checks update server every 24h or on WiFi connect
- Update server: GitHub Releases API (or custom CDN — TBD)

### Download & Install Flow

```
1. Update detected → ESP notifies RP → RP notifies PC app via USB

2. PC app shows actionable notification:
   "Firmware v1.2 available — [Release Notes] [Download & Install] [Later]"

3. User clicks [Download & Install]
   → PC signals board → ESP32 begins HTTP range requests in chunks
   → Each chunk: ESP → UART → RP → written to INACTIVE A/B partition
   → PC app shows live progress bar

4. All chunks received → SHA256 verified against server hash
   ├── Corrupt → partition abandoned, notify "Download failed, try again"
   └── Verified → metadata written:
       next_boot = INACTIVE, boot_attempts = 0, confirmed = FALSE

5. PC app shows: "Update ready! [Restart Now] [Later]"
   User clicks [Restart Now] → board reboots

6. Bootloader boots INACTIVE partition
   FreeRTOS health check:
   ├── Pass × 3 boots → confirmed = TRUE, old partition becomes standby ✅
   └── Fail × 3 boots → auto-rollback to OLD partition ✅

7. Power loss DURING write:
   → Metadata never changed → board boots OLD partition ✅

8. Power loss AFTER metadata write, BEFORE confirm:
   → boot_attempts increments each attempt
   → After 3 failures → auto-rollback ✅
```

> **Nothing can brick the board through OTA. The old partition is always the safety net.**

---

## WinUI3 PC App — Feature List

| Panel | Features |
|---|---|
| **Code Editor** | Syntax-highlighted Python editor, multi-file support |
| **File Manager** | USR_FC1 filesystem browser, drag-drop upload/delete |
| **Serial Monitor** | Timestamped output, filtering, copy |
| **Live Python REPL** | Interactive shell on board over USB CDC |
| **GPIO Dashboard** | Live pin state visualization, toggle digital outputs |
| **Visualization** | `pch matplotlib` renders here, live sensor graphs |
| **Library Manager** | Search + one-click install into bundled CPython venv |
| **Firmware Manager** | OTA notifications + progress, A/B swap, BC_FC3 recovery trigger |
| **Device Metrics** | Storage usage (USR_FC1 bar), CPU load, RAM usage, uptime |
| **WiFi Config** | SSID/password setup (written to ESP32 NVS, AES-256 encrypted) |

---

## Interactive Elements on Board

| Element | Count | Function |
|---|---|---|
| Digital GPIO (Female) | 21 | Programmable — digital, I2C, SPI, PWM, etc. |
| Analog GPIO (Female) | 8 | ADC input (A0–A7) |
| I2C Display Header | 1 | Plug-and-play display or other I2C device |
| Buzzer | 1 | User-accessible + system fault alert |
| RGB LED | 1 | System status indicator |
| Red LED | 1 | Power fault indicator (eFuse tripped) |
| Serial Debug Header | 1 | Advanced SWD/UART debugging |
| RST Tactile Switch | 1 | Board reset |
| USR_FLS Tactile Switch | 1 | 3-sec hold = mount USR_FC1 as USB drive |
| PWR-SEL Header (Male) | 1 | Jumper to select 5/9/12/20V output |
| VBUS_PROT Headers (Female) | 3 | Protected voltage output to user |
| SYS_R Pin Header | 1 | System restore trigger |
| FW_FLS Pin Header | 1 | Exposes RP_FC2 for manual firmware recovery |
