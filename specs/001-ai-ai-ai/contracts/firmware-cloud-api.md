# API合约: 固件 ↔ 云端服务

**版本**: v1.0 | **协议**: WebSocket (wss://) + HTTP | **日期**: 2025-10-07

## 概述

定义ESP32S3固件与云端AI服务之间的通信协议,包括WebSocket实时语音流和HTTP配置API。

---

## 1. WebSocket连接

### 连接端点

```
wss://api.lynxbox.com/ws/v1/voice
```

### 连接参数

Query参数:
- `device_id`: 设备唯一标识 (必填)
- `firmware_version`: 固件版本 (必填)
- `auth_token`: 设备认证Token (可选,未来实现)

示例:
```
wss://api.lynxbox.com/ws/v1/voice?device_id=lynx_abc12345_def678&firmware_version=v1.0.0
```

### 心跳机制

- 客户端每30秒发送WebSocket Ping帧
- 服务器响应Pong帧
- 超过2分钟无Ping则服务器主动断开连接

---

## 2. WebSocket消息格式

### 2.1 控制消息 (JSON)

#### 会话开始 (上行)

```json
{
  "type": "session_start",
  "session_id": "550e8400-e29b-41d4-a716-446655440000",
  "timestamp": 1234567890,
  "context": {
    "gesture": "shake",
    "battery_level": 85,
    "rssi": -65
  }
}
```

**字段说明**:
- `type`: 消息类型,固定值"session_start"
- `session_id`: UUID v4格式的会话ID
- `timestamp`: Unix时间戳(秒)
- `context.gesture`: 用户动作 (shake/flip/tap/null)
- `context.battery_level`: 电量百分比 (0-100)
- `context.rssi`: WiFi信号强度 (dBm)

#### 会话结束 (上行)

```json
{
  "type": "session_end",
  "session_id": "550e8400-e29b-41d4-a716-446655440000",
  "timestamp": 1234567900,
  "reason": "user_stopped"
}
```

**字段说明**:
- `reason`: 结束原因 (user_stopped / timeout / error)

#### 打断指令 (下行)

```json
{
  "type": "interrupt",
  "session_id": "550e8400-e29b-41d4-a716-446655440000",
  "timestamp": 1234567895
}
```

**用途**: 云端检测到用户打断(新的session_start),通知固件停止当前播放

#### 错误消息 (下行)

```json
{
  "type": "error",
  "session_id": "550e8400-e29b-41d4-a716-446655440000",
  "error_code": "ASR_TIMEOUT",
  "error_message": "语音识别超时",
  "timestamp": 1234567890
}
```

**错误码定义**:
- `ASR_TIMEOUT`: ASR服务超时
- `AI_SERVICE_ERROR`: AI服务不可用
- `TTS_FAILED`: TTS合成失败
- `INVALID_AUDIO`: 音频格式错误

---

### 2.2 音频数据 (Binary)

#### 上行音频帧 (固件 → 云端)

**格式**: 二进制,每帧结构如下

```
+--------+--------+--------+--------+-----------------------+
| Type   | SeqNum | SeqNum | Flags  | Opus Payload (变长)   |
| 1 byte | 2 bytes (BE)    | 1 byte | ~640 bytes            |
+--------+--------+--------+--------+-----------------------+
```

**字段说明**:
- `Type`: 0x01 (用户语音帧)
- `SeqNum`: 序列号 (Big Endian, 用于检测丢包)
- `Flags`: 标志位
  - Bit 0: 是否为语音活动(VAD)
  - Bit 1: 是否为最后一帧
  - Bit 2-7: 保留

**Opus参数**:
- 采样率: 16kHz
- 声道: 单声道
- 码率: 32kbps VBR
- 帧大小: 20ms (320 samples)
- 每帧大小: 约80-160字节

#### 下行音频帧 (云端 → 固件)

**格式**: 与上行相同

```
+--------+--------+--------+--------+-----------------------+
| Type   | SeqNum | SeqNum | Flags  | Opus Payload (变长)   |
| 0x02   | 2 bytes (BE)    | 1 byte | ~640 bytes            |
+--------+--------+--------+--------+-----------------------+
```

**字段说明**:
- `Type`: 0x02 (AI语音帧)
- Flags Bit 1: 当为1时表示TTS最后一帧,固件播放完毕后结束会话

---

## 3. HTTP REST API

### 3.1 设备注册

**端点**: `POST /api/v1/devices/register`

**请求体**:
```json
{
  "device_id": "lynx_abc12345_def678",
  "mac_address": "AA:BB:CC:DD:EE:FF",
  "firmware_version": "v1.0.0",
  "hardware_version": "hw_v1.0"
}
```

**响应** (200 OK):
```json
{
  "status": "registered",
  "device_id": "lynx_abc12345_def678",
  "created_at": 1234567890
}
```

### 3.2 健康检查

**端点**: `GET /api/v1/health`

**响应** (200 OK):
```json
{
  "status": "ok",
  "version": "v1.0.0",
  "services": {
    "asr": "online",
    "ai": "online",
    "tts": "online"
  }
}
```

---

## 4. 错误处理

### 4.1 重试策略 (FR-015b)

- API超时: 5秒
- 自动重试: 1次 (总计2次请求)
- 失败后固件播放提示音"服务暂时不可用"

### 4.2 WebSocket重连

- 断开后立即尝试重连
- 指数退避: 1s, 2s, 4s, 8s, 16s (最大)
- 最多重连10次,全部失败则播放网络错误提示音

---

## 5. 安全性

- **TLS 1.2+**: 所有WebSocket和HTTP通信使用wss://和https://
- **证书验证**: 固件验证服务器证书(内置CA根证书)
- **设备认证**: 未来实现基于设备证书的双向TLS认证

---

## 6. 性能要求

| 指标 | 目标值 |
| -------- | -------- |
| WebSocket连接建立 | <3秒 |
| 首帧音频到达(上行) | <200ms |
| AI回应首帧到达(下行) | <2秒 |
| 端到端延迟 | <2秒 (SC-002) |
| 打断检测响应 | <500ms (SC-003) |

---

**下一步**: 参考此合约实现固件WebSocket客户端和云端WebSocket服务器
