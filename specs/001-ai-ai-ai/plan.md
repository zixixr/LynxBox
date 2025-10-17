# Implementation Plan: AI语音交互盒子(LynxBox)

**Branch**: `001-ai-ai-ai` | **Date**: 2025-10-10 | **Spec**: [spec.md](./spec.md)
**Input**: Feature specification from `/specs/001-ai-ai-ai/spec.md`

**Note**: This template is filled in by the `/speckit.plan` command. See `.specify/templates/commands/plan.md` for the execution workflow.

## Summary

LynxBox is an AI-powered voice interaction module embedded in plush toys. The core system provides real-time AI conversation, dual-microphone interruption detection (barge-in), motion-sensing interaction via IMU, wireless connectivity (WiFi/BLE, optional 4G), and intelligent power management. The device operates as an embedded IoT system with ESP32S3 microcontroller, streaming audio to cloud-based ASR/TTS/AI services while maintaining strict privacy compliance (no local storage of user voice data).

## Technical Context

**Language/Version**: C (ESP-IDF framework, FreeRTOS-based)
**Primary Dependencies**: ESP-IDF v5.x, Espressif AEC (Acoustic Echo Cancellation), I2S audio driver, NVS (Non-Volatile Storage), WiFi/BLE stacks, TLS/SSL (mbedTLS), IMU driver (QMI8658), optional ML307 4G module driver
**Storage**: 4-8MB Flash (firmware + audio assets + NVS), NVS partition for WiFi credentials (AES-256 encrypted), no persistent user data
**Testing**: ESP-IDF unit tests (Unity framework), hardware-in-the-loop testing for audio/IMU, integration tests for cloud API communication
**Target Platform**: ESP32S3 (Xtensa LX7 dual-core @240MHz, 512KB SRAM, 4-8MB Flash), hardware versions: WiFi-only / WiFi+4G(ML307)
**Project Type**: Embedded IoT (single firmware image, cloud-connected)
**Performance Goals**: Voice response latency ≤2s end-to-end, barge-in detection ≤500ms, WiFi auto-switch ≤30s, IMU recognition accuracy ≥85%, battery life ≥6h (medium usage)
**Constraints**: Privacy-first (no voice data storage), TLS-encrypted communication, AEC-based dual-mic processing, 13 system prompts × 2 languages ≤2MB (OGG/Opus), NVS ≤512KB, circular logging ≤100KB, auto-sleep after 10min idle, wake-on-motion support
**Scale/Scope**: Single-device firmware, 10+ WiFi credential storage, multi-language support (CN/EN), OTA-ready architecture (future phase)

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

**Status**: ✅ PASSED (No constitution principles defined yet)

The project constitution (`.specify/memory/constitution.md`) is currently a template with no specific principles defined. Once the constitution is established, this section will be updated with specific compliance checks.

**Recommended Constitution Principles for Embedded IoT**:
- **Privacy-First Architecture**: No user data retention, encrypt all credentials
- **Resource Constraints**: Memory/Flash budgets must be validated before implementation
- **Reliability-First**: All error conditions must have defined fallback behavior
- **Hardware Abstraction**: Hardware-dependent code must be isolated in HAL layer
- **Testing Requirements**: Critical paths (audio, network, power) require hardware-in-loop tests

## Project Structure

### Documentation (this feature)

```
specs/[###-feature]/
├── plan.md              # This file (/speckit.plan command output)
├── research.md          # Phase 0 output (/speckit.plan command)
├── data-model.md        # Phase 1 output (/speckit.plan command)
├── quickstart.md        # Phase 1 output (/speckit.plan command)
├── contracts/           # Phase 1 output (/speckit.plan command)
└── tasks.md             # Phase 2 output (/speckit.tasks command - NOT created by /speckit.plan)
```

### Source Code (repository root)

```
firmware/
├── main/
│   ├── main.c                      # Entry point, FreeRTOS task initialization
│   ├── app_config.h                # Configuration constants (timeouts, thresholds)
│   ├── audio/
│   │   ├── audio_manager.c/h       # Audio pipeline orchestration
│   │   ├── mic_driver.c/h          # I2S dual-mic input + AEC integration
│   │   ├── speaker_driver.c/h      # I2S speaker output
│   │   ├── codec_opus.c/h          # Opus encoder/decoder wrapper
│   │   └── barge_in.c/h            # Barge-in detection logic (AEC + VAD)
│   ├── network/
│   │   ├── wifi_manager.c/h        # WiFi connection, multi-credential management
│   │   ├── ble_provisioning.c/h    # BLE pairing for WiFi provisioning
│   │   ├── lte_manager.c/h         # Optional ML307 4G module driver
│   │   └── http_client.c/h         # TLS/SSL HTTP client for cloud APIs
│   ├── cloud/
│   │   ├── ai_service.c/h          # WebSocket/HTTP API for ASR/TTS/AI
│   │   ├── config_sync.c/h         # Cloud-based device configuration (language, etc.)
│   │   └── device_report.c/h       # Device status reporting to cloud
│   ├── sensors/
│   │   ├── imu_driver.c/h          # QMI8658 IMU driver (I2C/SPI)
│   │   ├── motion_recognition.c/h  # Gesture detection algorithms (shake, flip, tap)
│   │   └── battery_monitor.c/h     # ADC-based battery level & temperature monitoring
│   ├── power/
│   │   ├── power_manager.c/h       # Sleep/wake state machine, charging logic
│   │   └── thermal_protection.c/h  # Over-temperature protection
│   ├── storage/
│   │   ├── nvs_manager.c/h         # NVS wrapper for WiFi credentials (encrypted)
│   │   └── assets_loader.c/h       # System audio assets loading from Flash partition
│   ├── ui/
│   │   ├── button_handler.c/h      # Button press detection (short/long/ultra-long)
│   │   ├── led_controller.c/h      # LED breathing/blinking patterns
│   │   └── audio_prompts.c/h       # System prompt playback (13 types × 2 languages)
│   ├── state/
│   │   ├── device_fsm.c/h          # Device state machine (boot, idle, active, sleep, etc.)
│   │   └── session_context.c/h     # Conversation session context (motion, timestamps)
│   └── utils/
│       ├── logging.c/h             # Circular logging (100KB buffer)
│       ├── crypto_utils.c/h        # AES-256 wrapper for credential encryption
│       └── watchdog.c/h            # System watchdog timer
├── components/                     # ESP-IDF components (if custom)
│   └── espressif_aec/              # Vendored Espressif AEC library (if needed)
├── partitions.csv                  # Flash partition table
├── sdkconfig.defaults              # ESP-IDF default configuration
├── CMakeLists.txt                  # Build configuration
└── Kconfig.projbuild               # menuconfig options (language, hardware variant)

test/
├── unit/                           # ESP-IDF Unity tests (on-target)
│   ├── test_audio_codec.c
│   ├── test_motion_recognition.c
│   └── test_nvs_manager.c
├── integration/                    # Hardware-in-loop tests
│   ├── test_wifi_provisioning.c
│   ├── test_cloud_api.c
│   └── test_power_management.c
└── mocks/                          # Mock drivers for simulation
    ├── mock_i2s.c
    └── mock_wifi.c

assets/
├── audio/
│   ├── cn/                         # Chinese system prompts (OGG/Opus, ≤1MB)
│   │   ├── boot.ogg
│   │   ├── shutdown.ogg
│   │   ├── wifi_success.ogg
│   │   └── [10 more prompts...]
│   └── en/                         # English system prompts (OGG/Opus, ≤1MB)
│       └── [same 13 prompts...]
└── certs/
    └── ca_bundle.pem               # Root CA certificates for TLS

docs/
└── [auto-generated Doxygen output]
```

**Structure Decision**: ESP-IDF embedded firmware structure with FreeRTOS-based task model. Key design decisions:
- **Layered architecture**: HAL (drivers) → Services (managers) → Application (FSM)
- **Hardware abstraction**: All hardware-specific code isolated in `audio/`, `sensors/`, `network/` drivers
- **Resource management**: Explicit memory budgets enforced via `heap_caps` allocations
- **Multi-language assets**: Separate asset partitions for CN/EN, switchable via build config or cloud sync

## Complexity Tracking

*Fill ONLY if Constitution Check has violations that must be justified*

| Violation | Why Needed | Simpler Alternative Rejected Because |
|-----------|------------|-------------------------------------|
| [e.g., 4th project] | [current need] | [why 3 projects insufficient] |
| [e.g., Repository pattern] | [specific problem] | [why direct DB access insufficient] |
