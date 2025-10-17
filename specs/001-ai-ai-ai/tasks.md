# Tasks: AI语音交互盒子 (LynxBox) - Firmware Implementation

**Feature Branch**: `001-ai-ai-ai`
**Input**: Design documents from `/specs/001-ai-ai-ai/`
**Prerequisites**: plan.md, spec.md, data-model.md, contracts/, research.md, quickstart.md

**Tests**: Hardware-in-loop testing required for embedded firmware. Unit tests (on-target) and integration tests included per plan.md recommendations.

**Organization**: Tasks organized by user story (P1-P5) to enable independent implementation and testing of each feature increment.

## Format: `[ID] [P?] [Story] Description`
- **[P]**: Can run in parallel (different files/modules, no dependencies)
- **[Story]**: User story mapping (US1, US2, US3, US4, US5, US6)
- Include exact file paths in descriptions

## Path Conventions
Based on plan.md firmware structure:
- **Firmware**: `firmware/main/` (C source + headers)
- **Tests**: `test/unit/`, `test/integration/`, `test/mocks/`
- **Assets**: `assets/audio/cn/`, `assets/audio/en/`
- **Build config**: `firmware/` (CMakeLists.txt, partitions.csv, sdkconfig.defaults)

---

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: ESP-IDF project initialization and build environment

- [ ] T001 Create firmware project structure per plan.md layout (firmware/main/, test/, assets/)
- [ ] T002 Initialize ESP-IDF project with CMakeLists.txt and component dependencies
- [ ] T003 Configure Flash partition table in `firmware/partitions.csv` (factory 2MB, audio_cn 1MB, audio_en 1MB, NVS 512KB, OTA reserves)
- [ ] T004 [P] Create sdkconfig.defaults with ESP32-S3 target, Flash size, TLS/mbedTLS, NVS encryption enabled
- [ ] T005 [P] Setup menuconfig options in `firmware/Kconfig.projbuild` for language selection (CONFIG_LANG_CN/EN), hardware variant (WiFi/4G)
- [ ] T006 [P] Create app_config.h with system constants (timeouts, thresholds, GPIO pin definitions, buffer sizes)
- [ ] T007 [P] Setup Unity test framework for on-target unit tests in `test/unit/`

**Checkpoint**: ESP-IDF project compiles successfully, can flash "Hello World" to ESP32-S3

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Core HAL drivers and system infrastructure that ALL user stories depend on

**⚠️ CRITICAL**: No user story work can begin until this phase is complete

### Hardware Abstraction Layer (HAL)

- [ ] T008 [P] Implement button handler driver in `firmware/main/ui/button_handler.c/h` (GPIO input, debounce, short/long/ultra-long press detection based on FR-001)
- [ ] T009 [P] Implement LED controller driver in `firmware/main/ui/led_controller.c/h` (GPIO PWM, breathing effect, blinking patterns per FR-005a)
- [ ] T010 [P] Implement I2S speaker driver in `firmware/main/audio/speaker_driver.c/h` (I2S master TX, PCM playback, DMA buffer management)
- [ ] T011 [P] Implement I2S dual-mic driver in `firmware/main/audio/mic_driver.c/h` (I2S master RX, dual-channel capture, 16kHz 16-bit)
- [ ] T012 [P] Implement QMI8658 IMU driver in `firmware/main/sensors/imu_driver.c/h` (I2C/SPI interface, read accel+gyro, interrupt configuration)
- [ ] T013 [P] Implement battery monitor driver in `firmware/main/sensors/battery_monitor.c/h` (ADC voltage reading, percentage calculation, temperature reading via NTC if available)

### Audio Processing

- [ ] T014 Integrate Espressif AEC component in `firmware/main/audio/mic_driver.c` (link esp-sr library, configure AEC for dual-mic echo cancellation per NFR-020)
- [ ] T015 Implement Opus codec wrapper in `firmware/main/audio/codec_opus.c/h` (encode/decode functions, 16kHz mono 32kbps, 20ms frames per NFR-022)
- [ ] T016 Implement audio manager in `firmware/main/audio/audio_manager.c/h` (pipeline orchestration, speaker/mic start/stop, buffer handling)

### Storage & Data Persistence

- [ ] T017 Implement NVS manager in `firmware/main/storage/nvs_manager.c/h` (init NVS, read/write key-value, enable Flash encryption per NFR-013)
- [ ] T018 Implement WiFi credential encryption in `firmware/main/utils/crypto_utils.c/h` (AES-256-CBC, PBKDF2 key derivation from eFuse MAC per data-model.md)
- [ ] T019 Implement asset loader in `firmware/main/storage/assets_loader.c/h` (mount SPIFFS audio partitions, load system prompts by ID)

### Network Stack

- [ ] T020 Implement WiFi manager in `firmware/main/network/wifi_manager.c/h` (WiFi STA mode init, connect, disconnect, event handlers)
- [ ] T021 Implement BLE provisioning in `firmware/main/network/ble_provisioning.c/h` (BLE GATT server, advertise "LynxBox-XXXX", receive WiFi credentials per contracts/app-firmware-ble.md)
- [ ] T022 Implement TLS HTTP client wrapper in `firmware/main/network/http_client.c/h` (esp_http_client + mbedTLS, certificate pinning)

### System Core

- [ ] T023 Implement device FSM in `firmware/main/state/device_fsm.c/h` (state machine: BOOT → IDLE → ACTIVE → SLEEP → OFF, state transitions per FR requirements)
- [ ] T024 Implement power manager in `firmware/main/power/power_manager.c/h` (sleep/wake logic, idle timeout 10min per FR-023, light sleep entry/exit)
- [ ] T025 Implement circular logging in `firmware/main/utils/logging.c/h` (100KB Flash buffer, wrap-around, sanitize sensitive data per NFR-018)
- [ ] T026 Implement watchdog wrapper in `firmware/main/utils/watchdog.c/h` (TWDT init, feed watchdog in main loop)
- [ ] T027 Create main.c entry point in `firmware/main/main.c` (app_main, FreeRTOS task creation, system initialization sequence)

**Checkpoint**: Foundation ready - all HAL drivers functional, system boots and enters IDLE state, LED indicators work

---

## Phase 3: User Story 1 - 基本语音对话交互 (Priority: P1) 🎯 MVP

**Goal**: User can power on device, connect to pre-configured WiFi, speak "你好", and receive AI voice response through speaker

**Independent Test**: Flash firmware with hardcoded WiFi credentials → Power on → Speak → Hear AI response within 2s (SC-002)

### Implementation for US1

- [ ] T028 [US1] Implement system audio prompts in `firmware/main/ui/audio_prompts.c/h` (load boot/shutdown/greeting prompts from assets, play via speaker_driver, priority queue per FR-018)
- [ ] T029 [US1] Implement barge-in detection in `firmware/main/audio/barge_in.c/h` (VAD post-AEC, detect user speech during playback, <500ms response per NFR-002)
- [ ] T030 [US1] Implement cloud WebSocket client in `firmware/main/cloud/ai_service.c/h` (wss:// connection, send/receive per contracts/firmware-cloud-api.md, binary frame parsing)
- [ ] T031 [US1] Implement session context manager in `firmware/main/state/session_context.c/h` (generate UUID session_id, attach motion/battery/RSSI context, send session_start/end messages)
- [ ] T032 [US1] Integrate voice interaction loop in `device_fsm.c` ACTIVE state (mic → Opus encode → WebSocket send → receive TTS → Opus decode → speaker play)
- [ ] T033 [US1] Implement motion recognition basic algorithms in `firmware/main/sensors/motion_recognition.c/h` (shake/flip/tap detection per NFR-021, attach to session context per FR-014)
- [ ] T034 [US1] Add cloud API retry logic in `ai_service.c` (2s timeout, 1 retry, fallback to "服务暂时不可用" prompt per FR-015b)
- [ ] T035 [US1] Add boot sequence in `main.c` (GPIO init → button long-press 3s detection → play boot音乐 → transition to IDLE per US1 acceptance criteria)
- [ ] T036 [US1] Add greeting playback after WiFi connected in `wifi_manager.c` event handler (fetch greeting from cloud config, play via audio_prompts per US1 AC2)

### Unit Tests for US1 (On-Target)

- [ ] T037 [P] [US1] Unit test for Opus codec in `test/unit/test_audio_codec.c` (encode/decode round-trip, verify MSE < threshold per plan.md example)
- [ ] T038 [P] [US1] Unit test for motion recognition in `test/unit/test_motion_recognition.c` (inject synthetic IMU data, verify shake/flip/tap detected)

### Integration Tests for US1 (Hardware-in-Loop)

- [ ] T039 [US1] Integration test for voice interaction in `test/integration/test_voice_interaction.c` (connect to mock cloud WebSocket, send audio frame, verify response received < 2s)

**Checkpoint**: US1 complete - User can power on, hear boot music, speak, and get AI response. MVP功能验证完成!

---

## Phase 4: User Story 2 - 初次配网设置 (Priority: P2)

**Goal**: User can provision device with WiFi credentials via BLE from mobile app, connect to WiFi automatically

**Independent Test**: Flash firmware → Power on (no WiFi configured) → Hear "请使用手机APP连接我" → Connect via BLE → Send SSID/password → Device connects to WiFi

### Implementation for US2

- [ ] T040 [US2] Implement multi-WiFi credential storage in `firmware/main/storage/nvs_manager.c` (save/load array of up to 10 WiFiCredential structs per data-model.md, encrypt passwords using crypto_utils)
- [ ] T041 [US2] Implement WiFi auto-selection in `wifi_manager.c` (scan周围AP, match saved SSIDs, sort by RSSI, connect to strongest per FR-010a)
- [ ] T042 [US2] Implement WiFi auto-switch in `wifi_manager.c` disconnect handler (detect signal loss, trigger scan, connect to next best saved WiFi within 30s per NFR-003)
- [ ] T043 [US2] Add BLE pairing mode entry in `device_fsm.c` BOOT state (if no saved WiFi credentials, transition to BLE_PAIRING, play "请使用手机APP连接我" per US2 AC1)
- [ ] T044 [US2] Add WiFi credential reception in `ble_provisioning.c` (parse SSID/password from BLE characteristic write, validate, save via nvs_manager, attempt connection)
- [ ] T045 [US2] Add WiFi connection success/fail audio prompts in `audio_prompts.c` (play "网络连接成功" or "网络连接失败,请重试" per FR-009)
- [ ] T046 [US2] Add BLE timeout logic in `device_fsm.c` BLE_PAIRING state (10min no connection → enter SLEEP, button wake re-enters pairing per edge case handling)
- [ ] T047 [US2] Implement WiFi credential management API in `nvs_manager.c` (add/delete/list credentials, support up to 10 networks per FR-010)

### Integration Tests for US2

- [ ] T048 [US2] Integration test for WiFi provisioning flow in `test/integration/test_wifi_provisioning.c` (simulate BLE write → verify WiFi connection → verify credential saved in NVS)
- [ ] T049 [US2] Integration test for WiFi auto-switch in `test/integration/test_wifi_provisioning.c` (save 2 credentials, disable AP1, verify switches to AP2 within 30s per US2 AC6)

**Checkpoint**: US2 complete - Device can be provisioned via BLE, connect to WiFi, auto-switch networks, manage up to 10 WiFi credentials

---

## Phase 5: User Story 3 - 电源管理 (Priority: P3)

**Goal**: Device monitors battery level, provides low-battery warnings, handles charging state, supports graceful shutdown

**Independent Test**: Monitor battery via ADC → Discharge to 15% → Hear "电量不足,请充电" every 5min → Connect charger → Hear "开始充电,充电期间请勿使用"

### Implementation for US3

- [ ] T050 [US3] Implement battery monitoring task in `battery_monitor.c` (periodic ADC read every 10s, calculate percentage per NFR-016, update DeviceState)
- [ ] T051 [US3] Implement low-battery warning in `power_manager.c` (detect <15%, trigger audio prompt every 5min per FR-019)
- [ ] T052 [US3] Implement critical battery shutdown in `power_manager.c` (detect <5%, play "电量耗尽,即将关机", transition to OFF per FR-020)
- [ ] T053 [US3] Implement charging detection in `battery_monitor.c` (detect USB VBUS via GPIO, set DeviceState.charging=true)
- [ ] T054 [US3] Implement charging audio prompts in `audio_prompts.c` + `power_manager.c` (play "开始充电,充电期间请勿使用" on charge start, "充电完成" at 100% per FR-021, FR-022)
- [ ] T055 [US3] Implement charging interaction lockout in `device_fsm.c` CHARGING state (disable voice/IMU, allow only button shutdown per FR-022a)
- [ ] T056 [US3] Implement thermal protection in `firmware/main/power/thermal_protection.c/h` (monitor battery temp >45°C, stop charge + play warning, auto-resume <40°C per FR-022b)
- [ ] T057 [US3] Implement shutdown sequence in `main.c` (detect button long-press 5s in IDLE/ACTIVE, play "再见,下次见" + shutdown音乐, power off per US3 AC4)

### Unit Tests for US3

- [ ] T058 [P] [US3] Unit test for battery percentage calculation in `test/unit/test_battery_monitor.c` (test voltage to % conversion, edge cases 4.2V=100%, 3.0V=0%)

**Checkpoint**: US3 complete - Battery monitoring, charging management, thermal protection, graceful shutdown all functional

---

## Phase 6: User Story 6 - 动作唤醒交互 (Priority: P3)

**Goal**: Device wakes from sleep when user picks up or shakes it, resumes WiFi, plays greeting

**Independent Test**: Device in SLEEP mode → Pick up device (IMU detects motion) → Device wakes within 2s → Hear "你回来啦,我们聊聊吧!"

### Implementation for US6

- [ ] T059 [US6] Implement auto-sleep logic in `power_manager.c` (detect 10min idle in any state, transition to SLEEP, disable WiFi/BLE/mic per FR-023)
- [ ] T060 [US6] Implement IMU wake-on-motion in `imu_driver.c` (configure QMI8658 motion interrupt, set threshold 0.3g duration 200ms per research.md)
- [ ] T061 [US6] Implement wake logic in `power_manager.c` (detect IMU interrupt or button press, exit light sleep, restore WiFi, transition to IDLE per FR-025a/b)
- [ ] T062 [US6] Add wake greeting in `device_fsm.c` SLEEP→IDLE transition (if WiFi reconnects, play "你回来啦,我们聊聊吧!" per US6 AC1; else play pairing prompt per US6 AC2)
- [ ] T063 [US6] Optimize WiFi reconnect speed in `wifi_manager.c` (test WiFi保持连接 vs 断开, measure power vs latency per FR-024 note)

### Integration Tests for US6

- [ ] T064 [US6] Integration test for motion wake in `test/integration/test_power_management.c` (enter SLEEP, trigger IMU interrupt, verify wake < 2s, WiFi reconnected per NFR-004)

**Checkpoint**: US6 complete - Device enters sleep after 10min idle, wakes on motion/button, greets user

---

## Phase 7: User Story 4 - 设备重置与恢复 (Priority: P4)

**Goal**: User can factory reset device by holding button 10s while off, clearing all WiFi credentials and config

**Independent Test**: Power off device → Hold button 10s → Hear "设备重置中,请稍候" → Hear "重置完成,设备已恢复出厂设置" → Power on → Device behaves like new (no saved WiFi)

### Implementation for US4

- [ ] T065 [US4] Implement factory reset detection in `button_handler.c` (detect ultra-long press 10s in OFF state per FR-001)
- [ ] T066 [US4] Implement reset procedure in `firmware/main/storage/nvs_manager.c` (erase all NVS namespaces: wifi_multi, device_info, config; keep firmware/assets intact per FR-026)
- [ ] T067 [US4] Add reset audio prompts in `audio_prompts.c` (play "设备重置中,请稍候" on reset start, "重置完成,设备已恢复出厂设置" on complete per FR-028)
- [ ] T068 [US4] Add reset flow in `main.c` (detect reset trigger → play prompts → call nvs_manager erase → auto shutdown per FR-027)

**Checkpoint**: US4 complete - Factory reset clears all user config, device returns to new state

---

## Phase 8: User Story 5 - 动作识别交互增强 (Priority: P5)

**Goal**: During conversation, device recognizes shake/flip/tap gestures and AI responds with contextual phrases

**Independent Test**: Start conversation → Shake device 3 times fast → AI says "哇,你在摇我,好好玩!" (motion context passed to cloud)

### Implementation for US5

- [ ] T069 [US5] Refine motion recognition algorithms in `motion_recognition.c` (implement all 9 gestures per NFR-021: pickup, fast/gentle shake, flip, tap, freefall, still, combo, reference Project Magic parameters)
- [ ] T070 [US5] Add gesture intensity/duration fields to session context in `session_context.c` (extend context JSON per contracts/firmware-cloud-api.md)
- [ ] T071 [US5] Implement idle pickup greeting in `device_fsm.c` IDLE state (detect "pickup" gesture after 5min idle, play proactive greeting per US5 AC4)
- [ ] T072 [US5] Add freefall detection response in `motion_recognition.c` (detect freefall per NFR-021, send to AI for "哎呀,我摔倒了,你没事吧?" response per FR-004a)

### Unit Tests for US5

- [ ] T073 [P] [US5] Unit test for advanced gesture recognition in `test/unit/test_motion_recognition.c` (test all 9 gestures, combo gestures, verify accuracy ≥85% per NFR-005)

**Checkpoint**: US5 complete - Rich motion interaction, AI responds contextually to gestures, enhances user experience

---

## Phase 9: Multi-Language Support (Priority: Cross-Cutting)

**Goal**: Device supports CN/EN system prompts, switchable via menuconfig or cloud config

**Independent Test**: Build with CONFIG_LANG_EN → Flash → Power on → Hear English boot music and prompts

### Implementation for Multi-Language

- [ ] T074 [P] Create Chinese system audio assets in `assets/audio/cn/` (13 prompts: boot.ogg, shutdown.ogg, wifi_success.ogg, wifi_failed.ogg, charge_start.ogg, charge_complete.ogg, battery_low.ogg, battery_critical.ogg, temp_high.ogg, reset_start.ogg, reset_complete.ogg, pair_request.ogg, service_error.ogg per FR-016, Opus 32kbps, total <1MB per NFR-023)
- [ ] T075 [P] Create English system audio assets in `assets/audio/en/` (same 13 prompts in English, Opus 32kbps, <1MB)
- [ ] T076 Implement language selection in `assets_loader.c` (check CONFIG_LANG_CN/EN at compile time, mount audio_cn or audio_en partition per FR-046)
- [ ] T077 Implement cloud language config in `firmware/main/cloud/config_sync.c/h` (GET /api/v1/devices/{id}/config, parse language field, download alternate language pack if needed per FR-047)
- [ ] T078 Implement hot language update in `assets_loader.c` + `config_sync.c` (download new OGG files via HTTPS, write to alternate audio partition, remount without reboot per FR-049)

**Checkpoint**: Multi-language complete - Device ships with default language, can switch via cloud config post-deployment

---

## Phase 10: Optional 4G Module Support (Priority: Future/Optional)

**Goal**: WiFi+4G hardware variant can fallback to 4G LTE when WiFi unavailable

**Independent Test**: WiFi+4G hardware → Disable WiFi → Device connects via 4G → Voice interaction works

### Implementation for 4G (Optional)

- [ ] T079 [4G] Implement ML307 4G driver in `firmware/main/network/lte_manager.c/h` (UART AT commands, PPPoS data mode per research.md)
- [ ] T080 [4G] Add network priority logic in `wifi_manager.c` + `lte_manager.c` (prefer WiFi, fallback to 4G if WiFi unavailable, switch back when WiFi available per spec 1.4)
- [ ] T081 [4G] Add eSIM profile management in `lte_manager.c` (AT commands for eSIM activation, deferred to field deployment)

**Note**: 4G support deferred to hardware availability, not blocking MVP

---

## Phase 11: Polish & Cross-Cutting Concerns

**Purpose**: Improvements affecting multiple user stories, system hardening

- [ ] T082 [P] Implement device status reporting in `firmware/main/cloud/device_report.c/h` (POST /api/v1/devices/{id}/status, send battery/network/uptime per data-model.md, for APP display)
- [ ] T083 [P] Add comprehensive error handling in all modules (validate inputs, handle malloc failures, log errors, avoid crashes)
- [ ] T084 [P] Security hardening: Enable Flash encryption in sdkconfig.defaults, verify TLS certificate pinning in http_client.c per NFR-011
- [ ] T085 [P] Optimize memory usage: Profile heap usage, reduce buffer sizes if possible, ensure <400KB RAM usage per ESP32-S3 512KB SRAM budget
- [ ] T086 [P] Power optimization: Measure current consumption in each state (ACTIVE/IDLE/SLEEP), tune sleep parameters to achieve 6h battery life per NFR-016
- [ ] T087 [P] Code cleanup: Remove debug prints, add Doxygen comments for API headers, follow ESP-IDF coding style
- [ ] T088 Run quickstart.md validation (follow quickstart.md steps, verify all examples compile and work)
- [ ] T089 [P] Generate API documentation via Doxygen in `docs/` (if Doxygen configured)

**Checkpoint**: Firmware polished, secure, optimized, ready for production testing

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: No dependencies - can start immediately
- **Foundational (Phase 2)**: Depends on Setup completion - **BLOCKS ALL user stories**
- **User Stories (Phase 3-8)**: All depend on Foundational (Phase 2) completion
  - **US1 (P1) - Phase 3**: MVP, must complete first
  - **US2 (P2) - Phase 4**: Depends on US1 (needs boot sequence, WiFi stack)
  - **US3 (P3) - Phase 5**: Can proceed after Foundational, integrates with US1 (power states)
  - **US6 (P3) - Phase 6**: Can proceed after US3 (uses sleep logic)
  - **US4 (P4) - Phase 7**: Can proceed after US2 (resets WiFi credentials)
  - **US5 (P5) - Phase 8**: Depends on US1 (enhances motion context in conversations)
- **Multi-Language (Phase 9)**: Can proceed after US1 (replaces placeholder audio prompts)
- **4G Module (Phase 10)**: Optional, can proceed after US2 (extends network stack)
- **Polish (Phase 11)**: Depends on all desired user stories completion

### User Story Dependencies (Critical Path)

```
Foundational (Phase 2) → US1 (P1) → US2 (P2) → US3 (P3) → US6 (P3) → US4 (P4) → US5 (P5)
                         ↓          ↓          ↓          ↓
                         MVP      Provisioning Power    Motion  Reset  Enhanced
                                                         Wake           Gestures
```

### Within Each User Story

- Tests (on-target unit + HW-in-loop integration) should be run after implementation
- HAL drivers before services
- Services before FSM integration
- Core implementation before edge cases
- Story complete and validated before next priority

### Parallel Opportunities

- **Phase 1**: T003-T007 all marked [P] can run in parallel
- **Phase 2**: T008-T013 (HAL drivers) can run in parallel, T017-T019 (storage) in parallel, T020-T022 (network) in parallel
- **Within US1**: T037-T038 (unit tests) can run in parallel
- **Phase 9**: T074-T075 (audio asset creation) can run in parallel
- **Phase 11**: Most polish tasks (T082-T089) can run in parallel

**Team Strategy**: After Foundational complete, 2-3 developers can work on US1-US3 in parallel with careful module isolation

---

## Parallel Example: Phase 2 Foundational HAL Drivers

```bash
# Launch all HAL driver implementations in parallel:
Task T008: "Implement button handler in firmware/main/ui/button_handler.c/h"
Task T009: "Implement LED controller in firmware/main/ui/led_controller.c/h"
Task T010: "Implement I2S speaker driver in firmware/main/audio/speaker_driver.c/h"
Task T011: "Implement I2S dual-mic driver in firmware/main/audio/mic_driver.c/h"
Task T012: "Implement QMI8658 IMU driver in firmware/main/sensors/imu_driver.c/h"
Task T013: "Implement battery monitor in firmware/main/sensors/battery_monitor.c/h"

# All 6 tasks create different files with no dependencies
```

---

## Implementation Strategy

### MVP First (US1 Only)

1. Complete Phase 1: Setup (T001-T007) → ESP-IDF project ready
2. Complete Phase 2: Foundational (T008-T027) → **CRITICAL** - HAL + system core ready
3. Complete Phase 3: US1 (T028-T039) → Basic voice interaction works
4. **STOP and VALIDATE**: Hardware-in-loop test US1 independently
   - Flash device with hardcoded WiFi
   - Power on → Hear boot music
   - Speak "你好" → Hear AI response < 2s
   - Verify barge-in works < 500ms
5. Deploy MVP firmware to pilot testers

**MVP Delivery**: ~40 tasks (T001-T039), estimated 4-6 weeks with 2 embedded engineers

### Incremental Delivery

1. **Foundation** (Phase 1-2) → System boots, LED blinks, IMU reads → 2 weeks
2. **+US1 (MVP)** → Voice interaction → Deploy/Demo → 3 weeks
3. **+US2** → WiFi provisioning via BLE → Deploy/Demo → 2 weeks
4. **+US3+US6** → Power management + motion wake → Deploy/Demo → 2 weeks
5. **+US4+US5** → Factory reset + enhanced gestures → Deploy/Demo → 1 week
6. **+Multi-Language** → CN/EN support → Deploy to international markets → 1 week
7. **+Polish** → Production hardening → Final release → 1 week

**Total Timeline**: ~12 weeks (3 months) for complete feature set

### Parallel Team Strategy

With 3 embedded engineers (post-Foundational):

- **Engineer A**: US1 (voice interaction) → US5 (enhanced gestures)
- **Engineer B**: US2 (WiFi provisioning) → US4 (factory reset)
- **Engineer C**: US3 (power mgmt) + US6 (motion wake) → Multi-Language

**Coordination**: Weekly integration points, shared HAL/FSM review, HW-in-loop test rig scheduling

---

## Hardware-in-Loop Test Setup

Per plan.md recommendations:

- **Test Rig Components**:
  - ESP32-S3-DevKitC-1 board
  - Dual I2S microphones (INMP441 or similar)
  - I2S speaker/amplifier (MAX98357A)
  - QMI8658 IMU breakout (or MPU6050 substitute for initial testing)
  - WiFi router (configurable SSIDs for multi-network tests)
  - Motion simulator: Manual shaking or rotary stage (optional, for automated US5 tests)
  - Battery emulator: Adjustable PSU to simulate 3.0V-4.2V for US3 tests

- **Test Automation**:
  - Python scripts via UART to trigger tests, parse Unity output
  - GPIO triggers to simulate button presses (long/ultra-long)
  - Mock cloud WebSocket server for US1 integration tests

---

## Notes

- **[P] tasks**: Different files/modules, no sequential dependencies within phase
- **[Story] labels**: Traceability to spec.md user stories
- **ESP-IDF specifics**: Use `idf.py build flash monitor` workflow, Unity test framework runs on-target
- **Hardware constraints**: Flash budget 16MB, RAM budget <400KB heap, battery life ≥6h
- **Privacy compliance**: Zero voice data storage enforced in code reviews (FR-030)
- **AEC dependency**: Espressif AEC library required for barge-in (NFR-020), verify license compatibility
- **Commit strategy**: Commit after each HAL driver or feature increment, tag after each user story checkpoint
- **Testing rigor**: Hardware-in-loop integration tests are NOT optional for embedded - required for US1/US2/US3/US6 validation

---

**Generated**: 2025-10-10 by `/speckit.tasks` command
**Total Tasks**: 89 (T001-T089)
**MVP Scope**: 39 tasks (T001-T039, Phase 1-3)
**Estimated MVP Duration**: 4-6 weeks with 2 embedded engineers
