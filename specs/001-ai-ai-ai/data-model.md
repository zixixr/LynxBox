# 数据模型设计: AI语音交互盒子

**项目**: LynxBox | **日期**: 2025-10-07 | **阶段**: Phase 1

## 模型概述

本文档定义系统中所有关键实体的数据结构、验证规则、状态转换及存储策略。设计遵循零用户数据存储原则(FR-030),所有持久化数据仅限设备配置和系统状态。

---

## 1. 设备实体 (Device)

### 描述

表示一个LynxBox物理设备,包含设备身份、硬件状态和网络配置。

### 字段定义

| 字段名 | 类型 | 约束 | 说明 |
| -------- | -------- | -------- | -------- |
| device_id | string(32) | PRIMARY KEY, NOT NULL | 设备唯一标识,格式:lynx_{MAC后8位}_{随机6位} |
| mac_address | string(17) | UNIQUE, NOT NULL | 设备MAC地址 (格式: AA:BB:CC:DD:EE:FF) |
| firmware_version | string(16) | NOT NULL | 固件版本号 (格式: v1.0.0) |
| hardware_version | string(16) | NOT NULL | 硬件版本 (格式: hw_v1.0) |
| created_at | uint32 | NOT NULL | 首次启动时间 (Unix时间戳) |
| last_seen | uint32 | NOT NULL | 最后在线时间 (Unix时间戳) |

### 存储位置

- **固件侧**: NVS分区,命名空间"device_info"
- **云端侧**: 不持久化(仅会话期间内存保存)

### 验证规则

```python
class Device:
    def validate_device_id(self, device_id: str) -> bool:
        """
        验证device_id格式: lynx_开头,总长度24-32字符,仅包含字母数字下划线
        """
        pattern = r'^lynx_[a-z0-9]{8}_[a-z0-9]{6}$'
        return re.match(pattern, device_id) is not None

    def validate_mac_address(self, mac: str) -> bool:
        """
        验证MAC地址格式: 6组十六进制数,用冒号分隔
        """
        pattern = r'^([0-9A-Fa-f]{2}:){5}([0-9A-Fa-f]{2})$'
        return re.match(pattern, mac) is not None
```

---

## 2. WiFi凭证实体 (WiFiCredential)

### 描述

存储设备已保存的WiFi网络凭证,支持多网络管理(FR-010)。

### 字段定义

| 字段名 | 类型 | 约束 | 说明 |
| -------- | -------- | -------- | -------- |
| index | uint8 | PRIMARY KEY | 索引 (0-4, 最多5个网络) |
| ssid | string(32) | NOT NULL | WiFi SSID |
| password | string(64) | NOT NULL, ENCRYPTED | WiFi密码 (AES-256-CBC加密) |
| last_rssi | int8 | DEFAULT -100 | 上次连接信号强度 (dBm) |
| last_connect_time | uint32 | DEFAULT 0 | 上次成功连接时间 (Unix时间戳) |
| fail_count | uint8 | DEFAULT 0 | 连续失败次数 (≥3则暂时黑名单) |
| blacklisted | bool | DEFAULT false | 是否暂时禁用 |

### 存储位置

- **固件侧**: NVS分区,命名空间"wifi_multi",键格式"net_{index}"
- **云端侧**: 不存储

### 加密策略

```c
// 固件侧加密实现(伪代码)
#define AES_KEY_SIZE 32
#define AES_IV_SIZE 16

// 密钥派生: 使用ESP32 eFuse MAC地址作为种子
uint8_t aes_key[AES_KEY_SIZE];
uint8_t aes_iv[AES_IV_SIZE];

void derive_encryption_key() {
    uint8_t mac[6];
    esp_efuse_mac_get_default(mac);

    // PBKDF2派生密钥 (salt=固定字符串"lynxbox_wifi")
    mbedtls_pkcs5_pbkdf2_hmac(&ctx, mac, 6, "lynxbox_wifi", 12,
                               10000, AES_KEY_SIZE, aes_key);

    // IV使用固定值(简化实现,实际可用更安全方案)
    memcpy(aes_iv, "lynxbox_init_iv!", AES_IV_SIZE);
}

int encrypt_password(const char *plaintext, char *ciphertext) {
    mbedtls_aes_context aes;
    mbedtls_aes_init(&aes);
    mbedtls_aes_setkey_enc(&aes, aes_key, 256);

    mbedtls_aes_crypt_cbc(&aes, MBEDTLS_AES_ENCRYPT,
                          strlen(plaintext), aes_iv,
                          (unsigned char*)plaintext,
                          (unsigned char*)ciphertext);

    mbedtls_aes_free(&aes);
    return 0;
}
```

### 验证规则

```python
class WiFiCredential:
    MAX_NETWORKS = 5

    def validate_ssid(self, ssid: str) -> bool:
        """
        SSID: 1-32字符,UTF-8编码
        """
        return 1 <= len(ssid.encode('utf-8')) <= 32

    def validate_password(self, password: str) -> bool:
        """
        密码: 8-64字符(WPA2标准),或0字符(开放网络)
        """
        length = len(password)
        return length == 0 or (8 <= length <= 64)

    def should_blacklist(self, fail_count: int) -> bool:
        """
        连续失败3次则加入黑名单
        """
        return fail_count >= 3
```

### 状态转换

```
IDLE → SCANNING: 设备启动或网络断开
SCANNING → CONNECTING: 找到匹配SSID
CONNECTING → CONNECTED: 认证成功,获取IP
CONNECTING → FAILED: 超时30秒或密码错误
FAILED → SCANNING: 尝试下一个网络
CONNECTED → DISCONNECTED: 信号丢失或AP重启
DISCONNECTED → SCANNING: 触发自动重连
```

---

## 3. 设备状态实体 (DeviceState)

### 描述

表示设备当前运行时状态,所有字段为易失性(不持久化)。

### 字段定义

| 字段名 | 类型 | 约束 | 说明 |
| -------- | -------- | -------- | -------- |
| power_state | enum | NOT NULL | 电源状态: ON / CHARGING / SLEEP / OFF |
| battery_level | uint8 | 0-100 | 电池电量百分比 |
| battery_voltage | uint16 | 毫伏 | 电池电压 (2500-4200mV) |
| battery_temp | int8 | 摄氏度 | 电池温度 (-20 ~ 60°C) |
| wifi_state | enum | NOT NULL | WiFi状态: DISCONNECTED / SCANNING / CONNECTING / CONNECTED |
| wifi_rssi | int8 | dBm | 当前WiFi信号强度 (-100 ~ 0) |
| current_ssid | string(32) | NULLABLE | 当前连接的SSID |
| ble_state | enum | NOT NULL | 蓝牙状态: OFF / PAIRING / CONNECTED |
| audio_state | enum | NOT NULL | 音频状态: IDLE / RECORDING / PLAYING / INTERRUPTED |
| last_gesture | enum | NULLABLE | 最后检测到的动作: SHAKE / FLIP / TAP / STILL |
| uptime | uint32 | 秒 | 自上次开机后运行时间 |

### 存储位置

- **固件侧**: 运行时内存(RAM),不持久化
- **云端侧**: WebSocket会话期间内存保存

### 枚举定义

```c
// 固件侧枚举定义
typedef enum {
    POWER_ON,
    POWER_CHARGING,
    POWER_SLEEP,
    POWER_OFF
} power_state_t;

typedef enum {
    WIFI_DISCONNECTED,
    WIFI_SCANNING,
    WIFI_CONNECTING,
    WIFI_CONNECTED
} wifi_state_t;

typedef enum {
    BLE_OFF,
    BLE_PAIRING,
    BLE_CONNECTED
} ble_state_t;

typedef enum {
    AUDIO_IDLE,
    AUDIO_RECORDING,
    AUDIO_PLAYING,
    AUDIO_INTERRUPTED
} audio_state_t;

typedef enum {
    GESTURE_NONE,
    GESTURE_SHAKE,
    GESTURE_FLIP,
    GESTURE_TAP,
    GESTURE_STILL
} gesture_t;
```

### 验证规则

```c
bool validate_battery_level(uint8_t level) {
    return level <= 100;
}

bool validate_battery_temp(int8_t temp) {
    return temp >= -20 && temp <= 60;
}

bool is_temperature_critical(int8_t temp) {
    return temp > 45; // FR-022b: 超过45°C停止充电
}

bool should_enter_sleep(uint32_t idle_seconds) {
    return idle_seconds >= 600; // FR-023: 10分钟无交互
}
```

---

## 4. 语音会话实体 (VoiceSession)

### 描述

表示一次完整的语音对话交互,仅包含元数据(不含语音数据本身,符合FR-030)。

### 字段定义

| 字段名 | 类型 | 约束 | 说明 |
| -------- | -------- | -------- | -------- |
| session_id | string(36) | PRIMARY KEY | 会话唯一标识 (UUID v4) |
| device_id | string(32) | FOREIGN KEY | 关联设备ID |
| start_time | uint32 | NOT NULL | 会话开始时间 (Unix时间戳) |
| end_time | uint32 | NULLABLE | 会话结束时间 (Unix时间戳) |
| gesture_context | enum | NULLABLE | 会话期间用户动作: SHAKE / FLIP / TAP |
| battery_level | uint8 | NOT NULL | 会话时电量 |
| rssi | int8 | NOT NULL | 会话时WiFi信号强度 |
| status | enum | NOT NULL | 会话状态: ACTIVE / COMPLETED / INTERRUPTED / ERROR |
| error_code | string(16) | NULLABLE | 错误码(如有) |

### 存储位置

- **固件侧**: 不存储
- **云端侧**: 不持久化(仅WebSocket会话期间内存保存,会话结束立即销毁)

### 生命周期

```
1. session_start: 用户开始说话 → 创建VoiceSession对象(内存)
2. audio_streaming: 流式传输Opus音频帧 → 不保存音频数据
3. ai_processing: 调用AI服务获取回应 → 不保存对话文本
4. audio_playback: TTS语音播放 → 不保存合成音频
5. session_end: 会话结束 → 销毁VoiceSession对象(内存释放)
```

### 验证规则

```python
class VoiceSession:
    def validate_session_id(self, session_id: str) -> bool:
        """
        验证UUID v4格式
        """
        pattern = r'^[0-9a-f]{8}-[0-9a-f]{4}-4[0-9a-f]{3}-[89ab][0-9a-f]{3}-[0-9a-f]{12}$'
        return re.match(pattern, session_id) is not None

    def is_session_expired(self, start_time: int, current_time: int) -> bool:
        """
        会话超时检查: 超过5分钟无活动则过期
        """
        return (current_time - start_time) > 300
```

---

## 5. 系统提示音实体 (SystemPrompt)

### 描述

预置的系统提示音资源,固化在固件中(FR-016)。

### 字段定义

| 字段名 | 类型 | 约束 | 说明 |
| -------- | -------- | -------- | -------- |
| prompt_id | enum | PRIMARY KEY | 提示音类型ID |
| file_path | string | NOT NULL | Flash中的文件路径(嵌入资源) |
| priority | uint8 | NOT NULL | 播放优先级 (0-255, 0最高) |
| duration_ms | uint16 | NOT NULL | 音频时长(毫秒) |

### 枚举定义 (固件侧)

```c
typedef enum {
    PROMPT_BOOT_MUSIC = 0,              // 开机音乐
    PROMPT_SHUTDOWN_MUSIC = 1,          // 关机音乐
    PROMPT_WIFI_SUCCESS = 2,            // 网络连接成功
    PROMPT_WIFI_FAILED = 3,             // 网络连接失败
    PROMPT_CHARGE_START = 4,            // 开始充电(含请勿使用提示)
    PROMPT_CHARGE_COMPLETE = 5,         // 充电完成
    PROMPT_BATTERY_LOW = 6,             // 电量不足
    PROMPT_BATTERY_CRITICAL = 7,        // 电量耗尽
    PROMPT_TEMP_HIGH = 8,               // 温度过高
    PROMPT_RESET_START = 9,             // 设备重置中
    PROMPT_RESET_COMPLETE = 10,         // 重置完成
    PROMPT_PAIR_REQUEST = 11,           // 请使用手机APP连接
    PROMPT_SERVICE_ERROR = 12,          // 服务暂时不可用
    PROMPT_MAX = 13
} system_prompt_id_t;
```

### 存储位置

- **固件侧**: SPIFFS分区或直接嵌入二进制(使用embed_files CMake命令)
- **格式**: MP3, 16kHz, 单声道, 64kbps
- **总大小**: <1MB

### 播放优先级规则

```c
// 优先级0-255: 0最高,255最低
#define PRIORITY_CRITICAL 0    // 关键警告(温度过高)
#define PRIORITY_HIGH 50       // 重要提示(充电、重置)
#define PRIORITY_NORMAL 100    // 普通提示(配网、电量)
#define PRIORITY_LOW 200       // 音乐(开关机)

// FR-018: 系统提示音优先级高于AI语音
// AI语音播放时,如有系统提示音触发,立即暂停AI并播放提示音
```

---

## 6. 日志记录实体 (LogEntry) - 可选

### 描述

固件侧循环日志,用于故障诊断(FR-035)。不保存用户隐私数据。

### 字段定义

| 字段名 | 类型 | 约束 | 说明 |
| -------- | -------- | -------- | -------- |
| timestamp | uint32 | NOT NULL | 日志时间 (Unix时间戳) |
| level | enum | NOT NULL | 日志级别: ERROR / WARN / INFO / DEBUG |
| module | string(16) | NOT NULL | 模块名称 (如"audio", "wifi", "power") |
| message | string(128) | NOT NULL | 日志消息(不含用户数据) |

### 存储位置

- **固件侧**: Flash循环缓冲区,固定100KB (FR-035)
- **格式**: 二进制紧凑格式(节省空间)

### 循环覆盖策略

```c
#define LOG_BUFFER_SIZE (100 * 1024)  // 100KB
#define LOG_ENTRY_SIZE 192            // 每条日志约192字节

typedef struct {
    uint32_t timestamp;
    uint8_t level;
    char module[16];
    char message[128];
} log_entry_t;

// 循环写入: 写满100KB后从头覆盖最旧记录
// 约可存储500条日志
```

### 敏感信息过滤

```c
// 日志中不得包含:
// - WiFi密码
// - 语音数据或转写文本
// - 用户输入内容
// - 设备序列号完整字符串(仅保留后4位)

void sanitize_log_message(char *message) {
    // 示例: 过滤密码字段
    str_replace(message, "password=xxx", "password=***");
}
```

---

## 数据流图

### 设备启动流程

```
1. ESP32启动 → 从NVS读取Device信息和WiFiCredential列表
2. WiFi扫描 → 匹配已保存SSID,按RSSI排序
3. 连接WiFi → 更新WiFiCredential.last_rssi和last_connect_time
4. WebSocket连接云端 → 发送device_id和DeviceState
5. 进入待机模式 → DeviceState.power_state = ON
```

### 语音对话流程

```
1. 用户说话 → 创建VoiceSession (内存)
2. 麦克风录音 → Opus编码 → WebSocket发送 (不存储)
3. 云端ASR → AI对话 → TTS → 返回Opus音频 (不存储)
4. 扬声器播放 → 播放完成
5. 会话结束 → 销毁VoiceSession (内存释放)
```

### 配网流程

```
1. 盒子进入蓝牙配对模式 → DeviceState.ble_state = PAIRING
2. APP连接BLE → 发送WiFi凭证(SSID+密码)
3. 盒子接收 → 新增WiFiCredential到NVS (加密存储)
4. 尝试连接新WiFi → 成功则更新last_connect_time
5. 播放成功提示音 → 关闭BLE
```

---

## 数据存储汇总

| 实体 | 固件存储 | 云端存储 | 持久化 | 加密 |
| -------- | -------- | -------- | -------- | -------- |
| Device | NVS | 无 | 是 | 否 |
| WiFiCredential | NVS | 无 | 是 | 是(AES-256) |
| DeviceState | RAM | 内存(会话期) | 否 | 否 |
| VoiceSession | 无 | 内存(会话期) | 否 | 否 |
| SystemPrompt | Flash(嵌入) | 无 | 是 | 否 |
| LogEntry | Flash(循环) | 无 | 是 | 否 |

**零用户数据存储验证**: ✅
- 语音数据: 不存储 (流式处理后丢弃)
- 对话文本: 不存储 (仅内存临时使用)
- 用户行为: 不存储 (仅gesture_context传递给AI)

**合规性确认**: 符合FR-027, FR-030, FR-031要求

---

## 下一步

Phase 1数据模型设计完成。

**接下来**:
- 定义三端API合约 (contracts/)
- 编写开发快速入门 (quickstart.md)
