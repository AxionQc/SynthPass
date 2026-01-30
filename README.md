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

TODO document stuff better :3
