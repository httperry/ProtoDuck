# ProtoDuck API Reference

Welcome to the ProtoDuck API documentation. This reference covers every module available to user code running on the ProtoDuck microcontroller.

---

## Quick Start

```python
# Pins just work — no import needed
D1.write(HIGH)
A0.read()
bus = I2C(D3, D4)

# Import modules as you need them
from protoduck import wifi, led, buzzer

wifi.connect()
resp = wifi.get("https://api.example.com/data")
print(resp.json())

# Async main auto-runs — no boilerplate
async def main():
    while True:
        led.set(A0.map(0, 255), 0, 0)
        await asyncio.sleep_ms(100)
```

---

## Always-Available Globals

These are injected by `boot.py` automatically. **Never import them.**

| Name | Type | Description |
|---|---|---|
| `D0` – `D20` | Pin objects | Digital GPIO pins |
| `A0` – `A7` | Pin objects | Analog GPIO pins (ADC) |
| `HIGH` | `1` | Digital high (3.3V) |
| `LOW` | `0` | Digital low (0V) |
| `INPUT` | const | Pin direction: input |
| `OUTPUT` | const | Pin direction: output |
| `PULLUP` | const | Enable internal pull-up resistor |
| `PULLDOWN` | const | Enable internal pull-down resistor |
| `I2C()` | constructor | Create I2C bus |
| `SPI()` | constructor | Create SPI bus |
| `PWM()` | constructor | Create PWM output |
| `UART()` | constructor | Create UART port |
| `asyncio` | module | Pre-imported async library |

---

## Modules

Each module has its own detailed documentation page:

| Module | Import | Description |
|---|---|---|
| [GPIO](gpio.md) | *(no import needed)* | Pin I/O, ADC, I2C, SPI, PWM, UART |
| [WiFi](wifi.md) | `from protoduck import wifi` | WiFi connection, HTTP, WebSocket, MQTT, sockets, hotspot |
| [Bluetooth](bluetooth.md) | `from protoduck import bluetooth` | BLE scanning, connecting, advertising, GATT server |
| [Zigbee](zigbee.md) | `from protoduck import zigbee` | IEEE 802.15.4 mesh networking |
| [LED](led.md) | `from protoduck import led` | Onboard RGB status LED control |
| [Buzzer](buzzer.md) | `from protoduck import buzzer` | Onboard buzzer/tone control |
| [System](system.md) | `from protoduck import system` | Board info, memory, storage, reset |
| [Thread](thread.md) | `from protoduck import thread` | True parallel execution via FreeRTOS tasks |
| [PCH Bridge](pch.md) | `import pch` or `import pchost` | Run PC-side Python libraries from board code |

---

## Important Notes

- **Pin objects are true globals.** All pins (`D0`–`D20`, `A0`–`A7`), peripherals, and constants (`HIGH`, `LOW`, etc.) are natively available in your script's global scope. You never need to import them or use a prefix.
- **`async def main()` auto-runs.** If your script defines an async `main()` function, the system calls `asyncio.run(main())` for you automatically.
- **RST button = soft reset.** It restarts your Python code only. WiFi, BLE, and USB connections stay alive.
- **MicroPython ≠ CPython.** This is MicroPython 1.24+ (Python 3.4+ syntax). For full CPython libraries, use the `pch` bridge.
