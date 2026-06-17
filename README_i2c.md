# M260 / M261 — I2C Access via G-Code

Adds two new M-codes that expose the configured I2C bus to G-code senders,
enabling runtime interaction with any I2C peripheral (sensors, expanders, etc.)
without firmware changes per device.

> Status: scan, probe, 1/2-byte write and read verified on hardware against an
> LIS3DH accelerometer (see [Typical Usage](#typical-usage--sensor-register-access)).

## Prerequisites

An `i2c0:` block must be present in `config.yaml`:

```yaml
i2c0:
  sda_pin: gpio.21
  scl_pin: gpio.22
  frequency: 100000
```

## Parameter words

Parameters use G-code **value words**, *not* axis words. FluidNC treats `A`/`B`/`C`
as rotary-axis words: a machine without those axes rejects them with error 20 before
the M-code runs. Value words are parsed regardless of axis count, so these commands
work on any machine.

| Word | Meaning |
|---|---|
| `P` | I2C address (7-bit) |
| `Q` | first data byte (write) |
| `R` | second data byte (write, optional) |
| `L` | byte count (read) |

All values are **decimal integers**.

## Commands

### M260 — Write / Scan / Probe

| Form | Effect |
|---|---|
| `M260` | Scan bus, list all responding addresses |
| `M260 P<addr>` | Probe single address, report ACK or NACK |
| `M260 P<addr> Q<byte>` | Write 1 byte to device |
| `M260 P<addr> Q<byte> R<byte2>` | Write 2 bytes (e.g. register address + value) |

### M261 — Read

| Form | Effect |
|---|---|
| `M261 P<addr> L<count>` | Read `count` bytes from device |

## Response Format

```
[MSG:INFO: I2CRESP B0 SCAN: 0x18 0x3C]        ; scan result
[MSG:INFO: I2CRESP B0 A0x18 ACK]               ; probe result
[MSG:INFO: I2CRESP B0 A0x18: 33 00 F0 ...]     ; read result (hex bytes)
```

The `I2CRESP` prefix makes responses machine-parseable by the sender.

## Typical Usage — Sensor Register Access

Device at address 24 (0x18), e.g. an ST accelerometer:

```
; 1. Identify device: set register pointer to WHO_AM_I (0x0F=15), then read 1 byte
M260 P24 Q15
M261 P24 L1
; → I2CRESP B0 A0x18: 33

; 2. Configure sensor: write register 0x20 (=32) value 0x57 (=87) — 100 Hz, all axes on
M260 P24 Q32 R87

; 3. Read accelerometer output: set pointer to 0x28|0x80=0xA8 (=168, auto-increment), read 6 bytes
M260 P24 Q168
M261 P24 L6
; → I2CRESP B0 A0x18: C0 01 40 F8 80 3F
;   = XL XH YL YH ZL ZH (little-endian, signed); here Z ≈ +1 g → gravity, sensor lying flat
```

## Compatibility

Modelled on Marlin's M260/M261 but adapted to FluidNC's parser. Marlin uses `A`
(address) and `B` (byte/count); those are rotary-axis words in FluidNC and are
rejected on machines without an A/B axis, so this implementation uses value words
(`P`/`Q`/`R`/`L`) instead. Further differences:

- **No byte buffer**: M260 executes immediately as a single I2C transaction.
  Writes are limited to 1–2 bytes (`Q`, optional `R`); Marlin buffers N bytes via `S1`.
- **Response format** differs (`I2CRESP …` vs. Marlin's `i2c-reply:`).
- Only I2C bus 0 is addressed; multi-bus support is not implemented.

## Implementation

Two files changed:

**`FluidNC/src/GCode.h`** — two new `NonModal` enum values:
```cpp
I2C_Write = 2000,  // M260
I2C_Read  = 2001,  // M261
```

**`FluidNC/src/GCode.cpp`**
- Parser: `case 260` / `case 261` map to the NonModal values via modal group MM10.
- Validation: `P`/`Q`/`R`/`L` words are consumed for I2C commands; the optional float
  words `P`/`Q`/`R` are set to `NAN` when absent so execution can detect presence.
- Execution: `I2C_Write` (scan / probe / write) and `I2C_Read` access `config->_i2c[0]`
  via the existing `I2CBus::read()` / `I2CBus::write()` API. Scan and probe use a 1-byte
  read (avoids an ESP-IDF 4.x limitation with zero-length writes in
  `i2c_master_write_to_device`).

## Known Limitations

- Multi-byte write is limited to 2 bytes (`Q` + `R`). For longer payloads the
  sender must issue multiple M260 commands — device auto-increment permitting.
- I2C operations are synchronous and blocking. Avoid during motion-critical sequences.
- Bus index is hardcoded to 0; machines with `i2c1:` cannot be targeted.
