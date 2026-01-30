# SynthPass - BLE Proximity Communication Protocol

A Bluetooth Low Energy (BLE) based proximity communication protocol for CH572Q microcontroller. SynthPass enables devices to discover each other, exchange proximity data, and perform "boop" interactions based on RSSI ranging.

## Features

- **Chip Unique ID**: Uses CH572Q hardware unique ID (64-bit) as device identifier
- **Custom BLE Protocol**: Structured message format with header and payload
- **Proximity Ranging**: RSSI-based distance detection between devices
- **Message Types**: Broadcast, proximity, boop, and data exchange
- **Single Channel**: Fixed BLE channel 37 for all communication
- **RSSI Calibration**: Reference RSSI values for hardware calibration
- **Collision Avoidance**: Randomized broadcast timing to prevent repeated collisions

## Hardware

- **MCU**: CH572Q (RISC-V 32-bit, 60 MHz)
- **Framework**: ch32v003fun
- **Clock**: 60 MHz from external HSE crystal (PLL)
- **LED**: PA11
- **Radio**: Integrated BLE 2.4 GHz radio
- **Flash**: 248 KB
- **RAM**: 12 KB

## SynthPass Protocol

### Device Identification

Each device gets a unique 32-bit UID derived from the chip's 64-bit hardware ID:

```c
uint64_t chip_uid = *(uint64_t*)(0x3F018);  // Read chip unique ID
synthpass_uid = (chip_uid >> 32) ^ (chip_uid & 0xFFFFFFFF);  // XOR to 32-bit
```

### Frame Structure

```c
typedef struct {
    uint8_t pdu;                    // PDU type (0x02)
    uint8_t length;                 // Payload length
    uint8_t mac[6];                 // Device identifier (":3:3:3")
    SynthPass_Message_T msg;        // Message payload
} SynthPass_Frame_T;

typedef struct {
    uint32_t sender_uid;            // Sending device UID
    uint8_t msg_type;               // Message type (see below)
    int8_t ref_rssi;                // Reference RSSI for calibration
    uint8_t data[];                 // Optional data payload
} SynthPass_Message_T;
```

### Message Types

| Type                      | Value | Description                                    |
| ------------------------- | ----- | ---------------------------------------------- |
| `SYNTHPASS_BROADCAST`     | 0x00  | Periodic broadcast to announce presence        |
| `SYNTHPASS_PROX`          | 0x01  | Proximity ranging message with peer UID & RSSI |
| `SYNTHPASS_PROX_DATA`     | 0x11  | Optional data sent with proximity message      |
| `SYNTHPASS_PROX_DATA_ACK` | 0x21  | Acknowledgment of received proximity data      |
| `SYNTHPASS_BOOP`          | 0x02  | Request to enter boop mode (close proximity)   |
| `SYNTHPASS_BOOP_DATA`     | 0x12  | Data exchange during boop                      |
| `SYNTHPASS_BOOP_DATA_ACK` | 0x22  | Acknowledgment of boop data                    |
| `SYNTHPASS_UNBOOP`        | 0x03  | Exit boop mode (distance increased)            |

### Communication Flow

1. **Discovery Phase**
   - Devices broadcast `SYNTHPASS_BROADCAST` messages periodically (1000ms)
   - Other devices receive and validate frame (PDU + MAC check)

2. **Proximity Phase**
   - Upon detecting broadcast, respond with `SYNTHPASS_PROX` message
   - Include sender UID and measured RSSI
   - Broadcast interval reduced to 200ms when peers detected

3. **Boop Phase** (Close Proximity)
   - When RSSI indicates very close range, send `SYNTHPASS_BOOP`
   - Broadcast interval reduced to 20ms during boop
   - Exchange data with `SYNTHPASS_BOOP_DATA` messages
   - Timeout after 3000ms without messages
   - Exit with `SYNTHPASS_UNBOOP` when distance increases

## Configuration

### Clock Settings

Located in [src/funconfig.h](src/funconfig.h):

```c
#define FUNCONF_USE_HSE           1                   // External crystal
#define CLK_SOURCE_CH5XX          CLK_SOURCE_PLL_60MHz
#define FUNCONF_SYSTEM_CORE_CLOCK 60000000            // 60 MHz
```

### Protocol Parameters

Located in [src/synthpass.h](src/synthpass.h):

```c
#define ACCESS_ADDRESS 0x8E89BED6        // BLE advertising access address
#define SYNTHPASS_CHANNEL 37             // Fixed BLE channel (2402 MHz)
#define SYNTHPASS_BROADCAST_PERIOD 1000  // ms between broadcasts (no peers)
#define SYNTHPASS_PROX_PERIOD 200        // ms between broadcasts (with peers)
#define SYNTHPASS_BOOP_PERIOD 20         // ms between broadcasts (during boop)
#define SYNTHPASS_RANDOM_DELAY 20        // Random jitter to avoid collisions
#define SYNTHPASS_TIMEOUT 3000           // Boop timeout duration
```

## Building and Flashing

```bash
# Build the project
pio run

# Upload to device
pio run --target upload

# Monitor serial output
pio device monitor --baud 115200
```

## Operation

### Startup Sequence

1. Initialize GPIO and configure LED (PA11)
2. Read chip unique ID from address 0x3F018
3. Generate 32-bit UID by XORing upper and lower 32-bits
4. Initialize BLE radio (0 dBm TX power)
5. LED blinks 5 times
6. Enter receive mode on channel 37

### Runtime Operation

1. **Continuous Listening**: Device stays in RX mode on channel 37
2. **Frame Validation**:
   - Check PDU type (must be 0x02)
   - Verify MAC identifier (":3:3:3")
3. **Frame Processing**:
   - Print received frame info (RSSI, PDU, length, MAC)
   - Extract sender UID and message type
   - Process message based on type

### Serial Console Output

```
Synthpass init! UID:a7:3f:e2:1d
RX'd! RSSI:-45 PDU:2 len:12 MAC:3a:3a:33:3a:33:3a
RX'd! RSSI:-52 PDU:2 len:12 MAC:3a:3a:33:3a:33:3a
```

## Technical Details

### BLE Specifications

- **PHY**: 1 Mbps (PHY_1M)
- **Access Address**: 0x8E89BED6 (standard BLE advertising)
- **Channel**: 37 only (2402 MHz)
- **TX Power**: 0 dBm
- **Max Packet Size**: 255 bytes (BLE limit)

### Frame Validation

All received frames must pass validation:

- PDU type must equal 0x02 (non-connectable advertising)
- MAC field must match ":3:3:3" (SYNTHPASS_MAC)

### Unique ID Extraction

The CH572Q stores a 64-bit unique ID at memory address 0x3F018. SynthPass reads this and creates a 32-bit UID:

```
UID = (chip_uid[63:32]) XOR (chip_uid[31:0])
```

## Development Notes

### Current Implementation Status

- ✅ Basic frame reception and validation
- ✅ Chip UID extraction and initialization
- ✅ iSLER radio interface
- ✅ Frame structure definitions
- ⏳ TX frame transmission (currently commented out)
- ⏳ Message type handling
- ⏳ RSSI-based proximity detection
- ⏳ Boop mode state machine
- ⏳ Data exchange protocol

### Next Steps

1. Implement periodic broadcast transmission
2. Add message type handling in `incoming_frame_handler()`
3. Implement RSSI-based proximity state machine
4. Add boop detection and timeout logic
5. Implement data exchange protocol

## Troubleshooting

### No Frames Received

- Verify external crystal is connected and oscillating
- Check that other device is transmitting on channel 37
- Ensure MAC identifier matches ":3:3:3"
- Verify access address is 0x8E89BED6

### Serial Output Not Showing

- Check baud rate is 115200
- Verify USB-UART adapter connection
- Ensure UART pins are configured in ch32v003fun

### LED Not Blinking

- Check PA11 pin connection
- Verify GPIO initialization in code
- Test with multimeter/oscilloscope

## License

This project uses the ch32v003fun framework. See framework documentation for licensing details.
