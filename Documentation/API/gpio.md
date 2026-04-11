# GPIO

Welcome to the magic of ProtoDuck's hardware API. We've eliminated the boilerplate so you can focus on building. All pins, peripheral constructors, and constants are seamlessly injected into your script's environment automatically—**no import statements required.**

```python
# Beautifully clean hardware control directly out of the box
D1.write(HIGH)
A0.read()
```

---

## Constants

Always available globally:

| Name | Description |
|---|---|
| `HIGH` | Digital high (3.3V) |
| `LOW` | Digital low (0V) |
| `INPUT` | Pin direction: input |
| `OUTPUT` | Pin direction: output |
| `PULLUP` | Internal pull-up resistor |
| `PULLDOWN` | Internal pull-down resistor |

---

## Digital Pins

Available: `D0` – `D20` (21 pins)

### Writing & Reading

```python
D1.write(HIGH)
D1.write(LOW)
D1.toggle()

state = D1.read()       # 0 or 1
```

### Pin Mode

```python
D1.mode(INPUT)
D1.mode(INPUT, PULLUP)
D1.mode(INPUT, PULLDOWN)
D1.mode(OUTPUT)
```

> Mode is auto-detected from first use — `write()` sets OUTPUT, `read()` sets INPUT. Only call `.mode()` if you need pull resistors.

---

## Events

All support **decorator syntax**.

### `@Dn.on_high`

Fires when pin goes HIGH (rising edge).

```python
@D1.on_high
def pressed():
    led.set_color("green")
```

### `@Dn.on_low`

Fires when pin goes LOW (falling edge).

```python
@D1.on_low
def released():
    led.off()
```

### `@Dn.on_click`

Fires after a complete press-release cycle (HIGH → LOW → HIGH, or LOW → HIGH → LOW depending on wiring). This is the "button was clicked" event.

```python
@D1.on_click
def clicked():
    buzzer.beep()
    D9.toggle()
```

Optional debounce (default 50ms):
```python
@D1.on_click(debounce=100)
def clicked():
    print("Clicked!")
```

### `@Dn.on_change`

Fires on any state change.

```python
@D5.on_change
def motion():
    print("Motion detected!")
```

### Aliases

For users familiar with electrical terms:
- `on_rise` = `on_high`
- `on_fall` = `on_low`

---

## Analog Pins

Available: `A0` – `A7` (8 pins, 12-bit ADC)

### Reading

```python
A0.read()           # raw int, 0–4095
A0.read(10)         # average of 10 readings

A0.readvolt()       # float voltage, 0.0–3.3
A0.readvolt(10)     # average of 10 voltage readings
```

> **Pro Tip:** Need to smooth out a noisy sensor? Simply pass the number of samples to instantly get a clean, hardware-averaged reading!

### Mapping

The `map()` function provides a quick, mathematical conversion of the current reading into your desired range. It returns a snapshot of the value right now.

```python
A0.map(0, 100)        # 0–4095 → 0–100 (percentage)
A0.map(0, 180)        # for a servo angle
A0.map(0.0, 3.3)      # same as readvolt()
A0.map(0, 255)        # for LED brightness

# Use it directly
out.duty(A0.map(0, 100))
```

### Streaming (Live Value)

`stream()` starts background sampling and returns a **live value** that always holds the latest reading. Just use it like a variable.

```python
live = A0.stream()          # raw value, updates in background
print(live)                 # 2048
print(live)                 # 2051 (newer reading)

live = A0.stream(0, 100)    # mapped live value (0–100)
print(live)                 # 50
print(live)                 # 51

live.stop()                 # stop background sampling
```

Customize sample rate:

```python
live = A0.stream(rate=200)         # 200 samples/sec (default: 100)
live = A0.stream(0, 100, rate=50)  # mapped, 50 samples/sec
```

---

## Peripherals

Global constructors — no pin method chaining needed.

### I2C

```python
bus = I2C(D3, D4)                 # SDA, SCL
bus = I2C(D3, D4, freq=400000)    # with custom frequency
```

**Methods:**

```python
bus.scan()                         # → [0x3C, 0x48, ...]
bus.writeto(0x3C, b'\x00\x01')
bus.readfrom(0x3C, 4)
bus.write_reg(0x3C, 0xAC, b'\x33')
bus.read_reg(0x3C, 0x00, 6)
bus.release()                      # free pins back to GPIO
```

**Device handle** (for cleaner multi-device code):

```python
bus = I2C(D3, D4)
sensor = bus.device(0x38)
display = bus.device(0x3C)

sensor.write_reg(0xAC, b'\x33\x00')
data = sensor.read_reg(0x00, 6)

display.writeto(b'\x00\x10\xFF')
```

**Context manager** (auto-releases pins when done):

```python
with I2C(D3, D4) as bus:
    bus.writeto(0x3C, b'\x00')
# D3 and D4 are free again
```

### SPI

```python
bus = SPI(D5, D6, D7, D8)         # CS, SCK, MOSI, MISO
bus = SPI(D5, D6, D7)             # CS, SCK, MOSI (no MISO, write-only)
```

**Methods:**

```python
bus.write(b'\x01\x02')
bus.read(4)
bus.transfer(out, inp)             # full duplex
bus.release()
```

### PWM

```python
out = PWM(D9)                      # default: 1kHz, 50% duty
out = PWM(D9, freq=500, duty=25)
```

**Methods:**

```python
out.duty(75)          # change duty %
out.freq(1000)        # change frequency
out.stop()            # stop, pin goes LOW
```

### UART

```python
port = UART(D10, D11)              # RX, TX
port = UART(D10, D11, baud=9600)   # custom baud rate
```

**Methods:**

```python
port.write(b"hello")
port.read(64)
port.readline()
port.any()              # bytes in buffer
port.release()
```

### Pin Locking

Once a pin is used in a peripheral, it's locked:

```python
bus = I2C(D3, D4)
D3.write(HIGH)       # ❌ PinModeError: D3 is in use as I2C SDA

bus.release()
D3.write(HIGH)       # ✅ works now
```

---

## Exceptions

| Exception | When |
|---|---|
| `PinModeError` | Using `write()`/`read()` on a pin locked to a peripheral |
| `PinConflictError` | Pin already assigned to another peripheral |
| `PinReleasedError` | Using a bus object after it was released |
| `ADCError` | Calling `.read()`/`.readvolt()` on a non-ADC digital pin |

---

## Full Example

```python
from protoduck import wifi, led, buzzer

# Hardware objects are ready to go natively!

@D1.on_click
def button():
    buzzer.beep()
    resp = wifi.get("https://api.example.com/data")
    print(resp.json())

live = A0.stream(0, 100)    # live percentage

bus = I2C(D3, D4)
sensor = bus.device(0x48)

async def main():
    while True:
        led.set(int(live * 2.55), 0, 0)    # brightness from A0
        temp = sensor.read_reg(0x00, 2)
        print(f"Light: {live}%  Temp: {temp}")
        await asyncio.sleep(1)
```
