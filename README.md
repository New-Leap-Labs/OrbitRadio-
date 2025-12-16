# Somaiya OrbitRadio

## APRS/AFSK Transceiver System

**STM32F4-based APRS packet radio system with Bell 202 AFSK modulation for amateur radio communication**

---

## Overview

This project implements a complete APRS (Automatic Packet Reporting System) transceiver using STM32F4 microcontroller with the following capabilities:

- Bell 202 AFSK modulation (1200 Hz mark, 2200 Hz space at 1200 baud)
- AX.25 packet protocol for amateur radio
- DRA818U radio module integration
- 4-bit R-2R DAC for audio generation
- UART packet reception with validation
- IWDG watchdog for system reliability

### Key Features

- Packet-triggered transmission (no auto-beacon)
- Robust input validation (ASCII-only payloads)
- Memory-optimized design (512-byte buffers)
- Watchdog protection against system hangs
- 9600 Hz sample rate audio generation
- Hardware-accelerated DAC output

---

## Hardware Requirements

### Microcontroller
- STM32F446RE Nucleo board (or compatible STM32F4 series)
- Upto 180 MHz system clock
- 512 KB Flash memory
- 128 KB SRAM + 4 KB backup SRAM


### Radio Module
- DRA818U/V UHF/VHF transceiver module
- Currently configured for UHF operation at 435.2480 MHz (default) or 435.2500 MHz (backup)
- Supports VHF operation as well (configurable when required)
- UART control interface at 9600 baud

### DAC Configuration (4-bit R-2R Ladder)

| STM32 Pin | DAC Bit | Resistor Value |
|-----------|---------|----------------|
| PA6       | MSB (bit 0) | R |
| PA4       | bit 1 | 2R |
| PA1       | bit 2 | 4R |
| PA0       | LSB (bit 3) | 6R |

**R-2R Network:** Use 1kΩ for R, 2kΩ for 2R  
**Output:** Connect DAC output to DRA818 microphone input via 10µF coupling capacitor

### Pin Configuration

- PA10 (USART1 RX): Packet data input at 115200 baud
- PC7 (PTT_UHF): Push-to-talk control for DRA818
- PA5 (LD2): Status LED (toggles on valid packet reception)

---

## Packet Structure

### DataPacket_t Format (260 bytes total)

```c
typedef struct {
    uint8_t command_type;      // 0x01 = Frequency Change, 0x02 = Telemetry
    uint8_t payload_length;    // 1-256 bytes
    uint8_t padding[2];        // Reserved (ignored)
    uint8_t payload[256];      // ASCII text payload
} DataPacket_t;
```

### Valid Command Types

- 0x01 - Frequency change command (currently triggers telemetry)
- 0x02 - Telemetry transmission

### Payload Validation Rules

**Accepted Characters:**
- Printable ASCII: 0x20-0x7E (space through tilde)
- Whitespace: \t (0x09), \n (0x0A), \r (0x0D)

**Rejected Packets:**
- Empty payloads (length = 0)
- Oversized payloads (length > 256)
- Invalid command types (not 0x01 or 0x02)
- Non-ASCII data (binary/control characters)

---

## APRS Packet Format

Generated APRS packets follow AX.25 UI-frame structure:

```
Flag | Dest Addr | Src Addr | Path | Control | PID | Info Field | FCS | Flag
0x7E | VU2CWN-0  | VU3CDI-5 | WIDE1-1,WIDE2-1 | 0x03 | 0xF0 | >Payload... | CRC | 0x7E
```

### Callsign Configuration

Edit the following definitions in `main.c`:

```c
#define SRC_CALL   "VU3CDI"    // Source callsign
#define SRC_SSID   5           // Source SSID
#define DST_CALL   "VU2CWN"    // Destination callsign
#define DST_SSID   0           // Destination SSID
#define PATH1_CALL "WIDE1"     // Digipeater path 1
#define PATH1_SSID 1
#define PATH2_CALL "WIDE2"     // Digipeater path 2
#define PATH2_SSID 1
```

---

## Getting Started

### 1. Clone Repository

```bash
git clone https://github.com/yourusername/OrbitRadio-APRS.git
cd OrbitRadio-APRS
```

### 2. Open in STM32CubeIDE

- Open STM32CubeIDE
- Navigate to File → Open Projects from File System
- Select the project folder
- Build the project (Ctrl+B)

### 3. Configure IWDG Watchdog

- Open `RX_Final.ioc`
- Navigate to Computing → IWDG
- Enable IWDG
- Set Prescaler: 64, Reload: 4095 (timeout approximately 8.2 seconds)
- Save and regenerate code

### 4. Flash to STM32

- Connect STM32 Nucleo via USB
- Click Run (green play button) or press F11

### 5. Hardware Connections

```
STM32 PA10 (RX) ←─── UART TX from sender device
STM32 PC7 (PTT) ───→ DRA818 PTT pin
STM32 DAC Out   ───→ DRA818 MIC (via 10µF capacitor)
DRA818 Audio    ───→ Radio antenna/speaker
```

---

## Usage

### Sending a Packet (Python Example)

```python
import serial
import struct

ser = serial.Serial('COM3', 115200)  # Adjust port

# Create packet
command_type = 0x02  # Telemetry
payload = b"Temperature: 25.3C, Humidity: 60%"
payload_length = len(payload)

# Pack packet (4-byte header + payload)
packet = struct.pack('<BB2s', command_type, payload_length, b'\x00\x00')
packet += payload.ljust(256, b'\x00')  # Pad to 256 bytes

# Send
ser.write(packet)
print(f"Sent {len(packet)} bytes")
ser.close()
```

### Expected System Behavior

1. PTT activates (PC7 goes HIGH)
2. AFSK tones generated via DAC (1200/2200 Hz)
3. Radio transmits APRS packet
4. PTT releases after transmission

### Transmission Time Calculation

```
Time (seconds) = (Packet bytes × 8 bits) / 1200 baud
Example: 75 bytes = (75 × 8) / 1200 = 0.5 seconds
```

---

## AFSK Technical Details

### Bell 202 Standard

- Mark (binary 1): 1200 Hz
- Space (binary 0): 2200 Hz
- Baud rate: 1200 baud
- Sample rate: 9600 Hz (Timer3 interrupt)

### Audio Generation Pipeline

```
AX.25 Encoder → AFSK Modulator → Timer3 ISR → 4-bit DAC → Analog Audio
   (ax25.c)       (afsk.c)      (9600 Hz)    (R-2R)      (DRA818)
```

### Timer3 Configuration

```c
Clock: 84 MHz APB1
Prescaler: 83  → 84MHz / 84 = 1 MHz
Period: 103    → 1MHz / 104 ≈ 9615 Hz ≈ 9600 Hz
```

At 9600 Hz sample rate:
- 1200 Hz tone = 8 samples per cycle
- 2200 Hz tone = 4.36 samples per cycle

---

## Watchdog Protection

### IWDG Configuration

- Timeout: approximately 8.2 seconds
- Prescaler: 64
- Reload value: 4095
- LSI Clock: 32 kHz

### Refresh Points

- Main loop: Every 10ms
- Packet processing start
- During AFSK transmission (multiple points)
- After long delays

**Purpose:** Automatically reset system if hang or crash occurs

---

## Memory Usage

| Resource | Size | Usage |
|----------|------|-------|
| Flash | 512 KB | Program code, constants |
| RAM | 96 KB | Runtime variables |
| AX.25 Buffer | 512 bytes | Packet encoding |
| RX Buffer | 260 bytes | UART reception |
| Process Buffer | 260 bytes | Packet processing |
| DAC Masks | 128 bytes | Precomputed GPIO values |

### Optimizations

- Reduced AX.25 buffer from 4096 to 512 bytes
- Minimized stack usage in string operations
- Precomputed DAC lookup tables

---

## Troubleshooting

### Packets Rejected

**Causes:**
- Invalid command type (must be 0x01 or 0x02)
- Payload contains non-ASCII characters
- Payload length = 0 or > 256
- UART framing errors

**Solution:** Verify packet structure and ensure clean ASCII text

### No Audio Output

**Symptoms:** PTT activates but no tones  
**Causes:**
- R-2R DAC not connected properly
- Wrong GPIO pins
- Timer3 not running

**Solution:** Verify DAC wiring and check Timer3 initialization

### Radio Not Transmitting

**Symptoms:** Audio plays but no RF output  
**Causes:**
- DRA818 not initialized
- Wrong frequency/configuration
- PTT not connected

**Solution:** Check USART6 connection to DRA818 and verify AT commands

### System Hangs or Resets

**Symptoms:** Unexpected resets  
**Causes:**
- Watchdog timeout (hung code)
- Missing HAL_IWDG_Refresh() calls

**Solution:** Increase IWDG timeout or add more refresh points

---

## Project Structure

```
RX_Final/
├── Core/
│   ├── Inc/
│   │   ├── main.h              # Pin definitions, function prototypes
│   │   ├── afsk.h              # AFSK generator declarations
│   │   ├── ax25.h              # AX.25 encoder declarations
│   │   └── stm32f4xx_it.h      # Interrupt handlers
│   └── Src/
│       ├── main.c              # Main application logic
│       ├── afsk.c              # AFSK audio generation
│       ├── ax25.c              # AX.25 packet encoding
│       ├── stm32f4xx_it.c      # ISR implementations
│       └── stm32f4xx_hal_msp.c # HAL initialization
├── Drivers/                     # STM32 HAL drivers
├── RX_Final.ioc                # STM32CubeMX configuration
└── README.md                   # This file
```

---

## Technical References

- [APRS Protocol Specification](http://www.aprs.org/doc/APRS101.PDF)
- [AX.25 Link Access Protocol](https://www.tapr.org/pdf/AX25.2.2.pdf)
- [Bell 202 Standard](https://en.wikipedia.org/wiki/Bell_202_modem)
- [DRA818 Datasheet](http://www.dorji.com/docs/data/DRA818U.pdf)
- [STM32F4 Reference Manual](https://www.st.com/resource/en/reference_manual/dm00031020.pdf)

---

## Authors

**Somaiya OrbitRadio Team**

Callsigns: VU3CDI, VU3LTQ, VU3DNW, VU3DUN, VU2CWN

Project: APRS/AFSK Transceiver for Amateur Radio

---

## Acknowledgments

- STMicroelectronics for STM32Cube HAL
- APRS/AX.25 protocol developers
- DRA818 module creators
- Amateur radio community

---

## Contact

For questions or support:
- Open an issue on GitHub
- Email: nli@somaiya.edu

---

**73 de Somaiya OrbitRadio Team**

*Last updated: December 2025*

