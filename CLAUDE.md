# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

MeshCore is a lightweight C++ library for multi-hop packet routing on embedded devices using LoRa radios. It enables decentralized mesh networks for off-grid communication without internet infrastructure.

## Build Commands

Build using PlatformIO. The project supports ESP32, nRF52, RP2040, and STM32 platforms.

```bash
# Build a specific target
pio run -e Heltec_v3_repeater

# Build with version info (used by CI)
FIRMWARE_VERSION=v1.0.0 ./build.sh build-firmware RAK_4631_repeater

# Build all firmwares of a type
./build.sh build-companion-firmwares
./build.sh build-repeater-firmwares
./build.sh build-room-server-firmwares

# Build firmwares matching a pattern
./build.sh build-matching-firmwares RAK_4631

# List all available build targets
pio project config | grep 'env:' | sed 's/env://'
```

## Architecture

### Core Library (`src/`)

- **Packet.h/cpp**: Fundamental transmission unit. Defines payload types (ADVERT, TXT_MSG, REQ/RESPONSE, ACK, GROUP, PATH, TRACE) and routing modes (FLOOD, DIRECT).
- **Dispatcher.h/cpp**: Low-level task managing radio I/O and packet queuing/scheduling. Provides `Radio`, `PacketManager`, and `MillisecondClock` abstractions.v- **Mesh.h/cpp**: Extends Dispatcher with mesh protocol logic. Handles packet routing, encryption, peer management, and provides virtual methods for application-specific handling.
- **Identity.h/cpp**: Cryptographic identity (Ed25519 keys) for nodes.
- **MeshCore.h**: Core constants and base abstractions (`MainBoard`, `RTCClock`).

### Helpers (`src/helpers/`)

- **BaseChatMesh**: Higher-level mesh with contact management, message handling, CLI support
- **radiolib/**: RadioLib wrapper implementations for SX1262, SX1276, etc.
- **bridges/**: RS232, ESP-NOW bridging implementations
- **sensors/**: Environmental sensor integrations
- **ui/**: Display drivers (SSD1306, ST7789, etc.), button handling

### Hardware Variants (`variants/`)

Each variant folder contains:
- `platformio.ini`: Build environments for that hardware (repeater, companion_radio_ble/usb/wifi, room_server, sensor)
- `target.h/cpp`: Hardware-specific radio init, pin definitions, board class
- `*Board.h`: Board abstraction (battery, manufacturer name, GPIO)

### Example Applications (`examples/`)

- **companion_radio/**: Client app for external devices (phone/web) via BLE/USB/WiFi
- **simple_repeater/**: Message relay node
- **simple_room_server/**: BBS-style shared posts server
- **simple_secure_chat/**: Terminal-based chat
- **simple_sensor/**: Environmental sensor node

## Adding a New Hardware Variant

1. Create `variants/<name>/` with `platformio.ini`, `target.h`, `target.cpp`, `*Board.h`
2. Define pin mappings and radio class in platformio.ini build_flags
3. Create build environments inheriting from base configs (esp32_base, nrf52_base, etc.)
4. Add source filters to include variant and example code

## Code Style Guidelines

- **No dynamic memory allocation** except during setup/begin functions
- Keep code concise and embedded-focused - avoid unnecessary abstraction layers
- Use `.clang-format` style: 2-space indent, 110 column limit, attach braces
- Do NOT retroactively reformat existing code - creates unnecessary diffs
- Submit PRs against the `dev` branch
- For impactful changes, open an Issue first to discuss approach

## Key Build Flags

Defined in root `platformio.ini`:
- `LORA_FREQ`, `LORA_BW`, `LORA_SF`: Radio parameters
- `MESH_DEBUG=1`: Enable debug logging
- `MESH_PACKET_LOGGING=1`: Log all packets
- `BRIDGE_DEBUG=1`: Bridge debug output
- `MAX_CONTACTS`, `MAX_GROUP_CHANNELS`: Storage limits
