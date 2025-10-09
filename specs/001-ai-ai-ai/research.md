# 技术研究与决策: AI语音交互盒子

**项目**: LynxBox | **日期**: 2025-10-07 | **阶段**: Phase 0

## 研究概述

本文档记录所有技术选型决策、理由及备选方案评估,解决plan.md中标记为"NEEDS CLARIFICATION"的技术问题。

---

## 决策 1: 云端服务技术栈

### 选定方案

**Python 3.11+ with FastAPI**

### 选择理由

1. **AI服务SDK支持最佳**
   - OpenAI官方Python SDK成熟稳定
   - 阿里云通义千问、讯飞星火均提供官方Python SDK
   - 国内AI服务主流SDK均优先支持Python

2. **开发效率高**
   - FastAPI提供自动API文档生成(OpenAPI/Swagger)
   - 异步支持(async/await)适合I/O密集型AI服务调用
   - Pydantic数据验证减少错误

3. **社区生态强**
   - 丰富的语音处理库(librosa, soundfile, pydub)
   - 成熟的WebSocket支持(FastAPI原生)
   - Docker部署方案成熟

4. **适合MVP快速迭代**
   - 代码量少,可读性强
   - 适合小团队快速开发

### 备选方案对比

| 方案 | 优势 | 劣势 | 评分 |
| -------- | -------- | -------- | -------- |
| **Python/FastAPI** | AI SDK支持最佳,开发速度快,异步性能好 | 执行速度不如Node.js/Go | ⭐⭐⭐⭐⭐ |
| Node.js/Express | JavaScript全栈,执行速度快 | AI SDK支持较弱,异步复杂度高 | ⭐⭐⭐ |
| Go/Gin | 性能最佳,部署简单 | AI SDK几乎无官方支持,开发周期长 | ⭐⭐ |

### 技术栈详细配置

```
语言: Python 3.11+
框架: FastAPI 0.104+
ASGI服务器: Uvicorn
AI SDK:
  - 阿里云DashScope SDK (主推,见决策3)
  - Fallback: OpenAI Python SDK
音频处理: pydub, soundfile
WebSocket: FastAPI原生支持
部署: Docker + Gunicorn/Uvicorn
监控: Prometheus + Grafana
日志: structlog
```

---

## 决策 2: 移动APP技术选型

### 选定方案

**微信小程序 (WXML + JavaScript)**

### 选择理由

1. **用户覆盖率最高**
   - 中国市场微信渗透率>90%
   - 无需安装,扫码即用
   - 适合玩具产品的用户群体

2. **蓝牙API支持完善**
   - 微信小程序提供完整的BLE API
   - 支持蓝牙设备扫描、连接、读写特征值
   - API文档完善,社区案例丰富

3. **开发成本最低**
   - 单一代码库,无需iOS/Android分别开发
   - 开发周期短(预计2-3周)
   - 无需App Store审核

4. **维护成本低**
   - 云端更新,用户无感知
   - 无需考虑多版本兼容

### 备选方案对比

| 方案 | 优势 | 劣势 | 评分 |
| -------- | -------- | -------- | -------- |
| **微信小程序** | 覆盖率高,开发快,蓝牙API完善 | 功能受限于微信,审核约束 | ⭐⭐⭐⭐⭐ |
| React Native | 跨平台,功能完整,可独立分发 | 开发周期长,需双平台测试 | ⭐⭐⭐ |
| 原生iOS+Android | 性能最佳,功能无限制 | 开发成本高(2个团队),周期长 | ⭐⭐ |
| Flutter | 跨平台性能好,UI一致性强 | 生态不如RN,蓝牙插件成熟度低 | ⭐⭐ |

### 技术栈详细配置

```
框架: 微信小程序原生开发
语言: JavaScript (ES6+)
UI库: WeUI (微信官方UI组件库)
蓝牙: wx.openBluetoothAdapter, wx.createBLEConnection
状态管理: 原生Page data (无需引入Redux等)
网络: wx.request (RESTful API)
开发工具: 微信开发者工具
测试: 微信小程序自动化测试框架
最低版本要求: 微信7.0+
```

### 未来扩展路径

如未来需要独立APP:
1. Phase 2: 开发React Native版本
2. 复用小程序业务逻辑(JavaScript)
3. 通过App内嵌WebView快速过渡

---

## 决策 3: AI服务提供商

### 选定方案

**阿里云通义千问 (Qwen) + 讯飞星火语音**

组合方案:
- **ASR(语音识别)**: 讯飞星火实时转写API
- **AI对话**: 阿里云通义千问Qwen-Turbo
- **TTS(语音合成)**: 讯飞星火语音合成API

### 选择理由

1. **中文识别准确率最高**
   - 讯飞ASR在中文领域准确率>95%
   - 语音识别质量优秀
   - 支持方言和不标准发音

2. **延迟与成本平衡**
   - 通义千问Qwen-Turbo响应延迟<800ms
   - 讯飞实时转写延迟<500ms
   - 成本: ASR ¥0.003/次, AI ¥0.002/千token, TTS ¥0.002/次
   - 预估单次对话成本<¥0.02

3. **合规性强**
   - 阿里云、讯飞均为国内服务商,数据不出境
   - 符合《个人信息保护法》等隐私保护规定
   - 提供企业级SLA保障

4. **应用友好特性**
   - 讯飞提供多种音色(可爱、活泼、温柔等)
   - 通义千问支持角色扮演(可定制玩具人格)
   - 支持内容安全过滤(屏蔽不良内容)

### 备选方案对比

| 方案 | 中文准确率 | 延迟 | 成本(单次) | 合规性 | 评分 |
| -------- | -------- | -------- | -------- | -------- | -------- |
| **讯飞+通义千问** | 95%+ | <1.5s | ¥0.02 | ✅国内 | ⭐⭐⭐⭐⭐ |
| OpenAI Whisper+GPT-4 | 90% | <2s | ¥0.15 | ⚠️出境 | ⭐⭐⭐ |
| Azure认知服务 | 92% | <1.8s | ¥0.08 | ⚠️出境 | ⭐⭐⭐ |
| 腾讯云AI | 94% | <1.6s | ¥0.03 | ✅国内 | ⭐⭐⭐⭐ |

### 技术栈详细配置

```
ASR服务:
  - 提供商: 讯飞开放平台
  - 产品: 实时语音转写WebSocket API
  - 采样率: 16kHz, 16bit, 单声道
  - 语言: 中文(普通话)
  - 优化模式: 标准

AI对话服务:
  - 提供商: 阿里云百炼平台
  - 模型: Qwen-Turbo (7B参数)
  - API: DashScope SDK
  - 最大Token: 2048
  - 温度: 0.7-0.9 (活泼)
  - 系统提示词: 玩具角色人设

TTS服务:
  - 提供商: 讯飞开放平台
  - 产品: 在线语音合成API
  - 音色: 可选多种音色(活泼、温柔等)
  - 语速: 中速(speed=50)
  - 音量: 标准(volume=50)
  - 格式: MP3, 16kHz, 64kbps
```

### Fallback策略

主服务故障时切换顺序:
1. 主: 讯飞ASR + 通义千问 + 讯飞TTS
2. 备1: 腾讯云ASR + 腾讯混元 + 腾讯TTS
3. 备2: OpenAI Whisper + GPT-3.5 + OpenAI TTS

---

## 决策 4: 音频编解码方案

### 选定方案

**Opus编码 (16kHz, 单声道, 32kbps)**

### 选择理由

1. **压缩率与质量平衡最佳**
   - 32kbps码率下语音质量接近64kbps AAC
   - 带宽消耗降低50%,适合WiFi不稳定场景

2. **延迟最低**
   - Opus设计目标即低延迟实时通信
   - 帧大小可配置(20ms-60ms),本项目选20ms
   - 编解码延迟<5ms

3. **ESP32S3软件解码性能可接受**
   - Opus有优化的ARM Cortex实现(ESP32S3可复用)
   - 解码单帧(<1ms CPU时间@240MHz)
   - 内存占用<50KB

4. **开源且免费**
   - BSD许可,商业友好
   - 社区支持强(WebRTC标准编解码器)

### 备选方案对比

| 方案 | 压缩率 | 质量 | 延迟 | ESP32支持 | 评分 |
| -------- | -------- | -------- | -------- | -------- | -------- |
| **Opus** | 32kbps | 优秀 | <20ms | 软解,性能好 | ⭐⭐⭐⭐⭐ |
| AAC-LC | 64kbps | 优秀 | ~50ms | 软解,性能中 | ⭐⭐⭐ |
| ADPCM | 32kbps | 中等 | <10ms | 硬解,性能最佳 | ⭐⭐⭐⭐ |
| PCM(raw) | 256kbps | 完美 | 0ms | 无需解码 | ⭐⭐ (带宽太高) |

### 技术栈详细配置

```
编码器(ESP32S3):
  - 库: libopus (ESP-IDF component)
  - 采样率: 16kHz
  - 声道: 单声道
  - 码率: 32kbps (可变码率VBR)
  - 帧大小: 20ms (320 samples)
  - 复杂度: 5 (平衡质量与性能)

解码器(云端):
  - Python: opuslib / av
  - 转换目标: PCM 16kHz 16bit (供AI服务)

传输格式:
  - WebSocket: 二进制帧
  - 每帧: 20ms音频 + 4字节头(timestamp)
  - 每秒50帧 ≈ 4KB/s带宽
```

### ADPCM备选理由

如Opus解码性能不满足需求,可降级至ADPCM:
- 质量稍差但可接受
- ESP32S3硬件加速
- 实现最简单

---

## 决策 5: 通信协议架构

### 选定方案

**WebSocket (全双工) + HTTP (配置)**

- **实时语音交互**: WebSocket (wss://)
- **设备配置/管理**: HTTP/RESTful API
- **心跳保活**: WebSocket Ping/Pong (30秒间隔)

### 选择理由

1. **满足打断检测实时性要求**
   - WebSocket全双工,云端可主动推送停止指令
   - 延迟<100ms,满足500ms打断响应要求

2. **资源消耗低于MQTT**
   - 无需MQTT Broker额外部署
   - WebSocket直连云端,减少一跳
   - ESP32S3内存占用<MQTT客户端

3. **云端实现简单**
   - FastAPI原生WebSocket支持
   - 无需额外消息队列
   - 适合MVP快速上线

4. **适合流式音频传输**
   - 二进制帧传输,无Base64编码开销
   - 支持分片传输(大TTS音频)

### 备选方案对比

| 方案 | 双向通信 | 延迟 | 资源消耗 | 云端支持 | 评分 |
| -------- | -------- | -------- | -------- | -------- | -------- |
| **WebSocket** | ✅全双工 | <100ms | 低 | FastAPI原生 | ⭐⭐⭐⭐⭐ |
| HTTP/2 Server Push | ⚠️半双工 | ~200ms | 中 | 需Nginx配置 | ⭐⭐⭐ |
| MQTT | ✅全双工 | ~150ms | 高(需Broker) | 需部署Broker | ⭐⭐⭐ |
| HTTP轮询 | ❌单向 | >500ms | 极高 | 简单 | ⭐ |

### 协议设计

#### WebSocket消息格式 (JSON + Binary混合)

**控制消息(JSON)**:
```json
{
  "type": "session_start",
  "device_id": "lynx_xxxxx",
  "timestamp": 1234567890,
  "context": {
    "gesture": "shake",
    "battery": 85
  }
}
```

**音频数据(Binary)**:
```
[Header 4 bytes: type(1) + seq(2) + flags(1)]
[Payload: Opus frame]
```

**类型定义**:
- `0x01`: 用户语音帧(上行)
- `0x02`: AI语音帧(下行)
- `0x03`: 停止播放指令
- `0x04`: 会话结束

#### HTTP RESTful API

```
POST /api/v1/devices/register        # 设备注册
GET  /api/v1/devices/{id}/status     # 设备状态
POST /api/v1/devices/{id}/config     # 更新配置
GET  /api/v1/health                  # 健康检查
```

### 技术栈详细配置

```
ESP32S3 WebSocket客户端:
  - 库: esp_websocket_client (ESP-IDF)
  - TLS: mbedTLS (wss://加密)
  - 缓冲区: 4KB发送 + 8KB接收
  - 心跳: 30秒Ping

云端WebSocket服务器:
  - 框架: FastAPI WebSocket
  - 并发: 10,000连接 (Uvicorn workers=4)
  - 超时: 5分钟无数据自动断开
```

---

## 决策 6: 多WiFi管理方案

### 选定方案

**自研多WiFi管理器 (基于ESP-IDF WiFi组件扩展)**

### 选择理由

1. **ESP-IDF原生WiFi Manager不支持多网络**
   - 官方esp_wifi仅支持单SSID存储
   - 需要自研实现多网络管理

2. **NVS存储方案成熟**
   - 使用NVS(Non-Volatile Storage)存储加密凭证
   - 支持key-value存储,适合5个网络上限
   - Flash占用<4KB

3. **自动切换逻辑可控**
   - 信号强度优先(RSSI排序)
   - 失败重试策略可定制
   - 支持黑名单(连续3次失败暂时禁用)

4. **Flash占用可控**
   - 预计代码<2KB
   - 数据<4KB (5个网络 × 约200字节)

### 实现方案

#### 数据结构

```c
#define MAX_WIFI_NETWORKS 5
#define WIFI_SSID_MAX_LEN 32
#define WIFI_PASSWORD_MAX_LEN 64

typedef struct {
    char ssid[WIFI_SSID_MAX_LEN];
    char password[WIFI_PASSWORD_MAX_LEN]; // AES加密存储
    int8_t last_rssi;                     // 上次连接信号强度
    uint32_t last_connect_time;           // Unix时间戳
    uint8_t fail_count;                   // 连续失败次数
    bool blacklisted;                     // 是否暂时禁用
} wifi_credential_t;

typedef struct {
    wifi_credential_t networks[MAX_WIFI_NETWORKS];
    uint8_t count;
    int8_t current_index;
} wifi_manager_t;
```

#### 核心算法

**连接顺序**:
1. 扫描周围WiFi (esp_wifi_scan_start)
2. 与已保存网络匹配,按RSSI排序
3. 尝试连接信号最强且未被黑名单的网络
4. 失败则尝试下一个(超时30秒)
5. 全部失败则进入蓝牙配对模式

**自动切换触发**:
- 当前WiFi断开事件 (WIFI_EVENT_STA_DISCONNECTED)
- RSSI低于-80dBm持续30秒
- 定期扫描(每5分钟)发现更强信号(RSSI差>10dBm)

### 技术栈详细配置

```
组件结构:
  - components/wifi_multi/
    ├── wifi_multi.h          # API接口
    ├── wifi_multi.c          # 核心逻辑
    ├── wifi_storage.c        # NVS存储封装
    └── CMakeLists.txt

NVS命名空间: "wifi_multi"
NVS键前缀: "net_{index}" (net_0, net_1, ...)

加密:
  - 使用mbedTLS AES-256-CBC
  - 密钥派生自ESP32 eFuse MAC地址(设备唯一)

事件处理:
  - WiFi事件组(event group)
  - 状态机: IDLE → SCANNING → CONNECTING → CONNECTED → DISCONNECTED
```

---

## 决策 7: 固件OTA升级方案

### 选定方案

**暂不实现,标记为未来增强(Phase 2+)**

### 理由

1. **MVP不需要OTA**
   - 初期设备量小(<1000台)
   - 可通过USB烧录更新(内测阶段)

2. **复杂度较高**
   - 需要安全的OTA服务器
   - Flash分区需要预留OTA空间(双分区,占用2MB+)
   - 回滚机制、签名验证需要额外开发

3. **优先级低于核心功能**
   - 先保证语音对话、配网、电源管理稳定
   - OTA可在后续版本加入

### 未来实现路径 (参考)

当需要OTA时,推荐方案:
- **ESP-IDF原生OTA** (app_update组件)
- **双分区设计**: Factory + OTA_0 + OTA_1
- **HTTPS下载**: 从云端拉取固件(签名验证)
- **回滚机制**: 升级失败自动回滚到旧版本
- **灰度发布**: 先推送10%设备测试

预估Flash分区(8MB):
```
Factory:   1.5MB  (出厂固件)
OTA_0:     2.0MB  (运行固件A)
OTA_1:     2.0MB  (运行固件B)
NVS:       24KB   (配置存储)
Audio:     1.5MB  (系统提示音)
Others:    1.0MB  (其他)
```

---

## 研究结论汇总

| 决策项 | 选定方案 | 关键理由 |
| -------- | -------- | -------- |
| 云端技术栈 | Python/FastAPI | AI SDK支持最佳,开发效率高 |
| 移动APP | 微信小程序 | 覆盖率高,蓝牙API完善,开发成本低 |
| AI服务商 | 讯飞+通义千问 | 中文准确率高,合规性强,成本合理 |
| 音频编解码 | Opus 32kbps | 压缩率质量平衡,延迟低 |
| 通信协议 | WebSocket+HTTP | 全双工低延迟,实现简单 |
| 多WiFi管理 | 自研NVS方案 | 官方不支持,自研可控 |
| OTA升级 | 暂不实现 | MVP不需要,Phase 2+考虑 |

## 下一步行动

Phase 0研究完成,已解决所有"NEEDS CLARIFICATION"问题。

**进入Phase 1**:
1. 基于研究结论设计数据模型 (data-model.md)
2. 定义三端API合约 (contracts/)
3. 编写开发快速入门 (quickstart.md)

所有技术选型已确定,可开始详细设计阶段。
