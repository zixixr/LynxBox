# 实现计划: AI语音交互盒子(LynxBox)

**分支**: `001-ai-ai-ai` | **日期**: 2025-10-07 | **规格**: [spec.md](spec.md)
**输入**: 功能规格说明来自 `/specs/001-ai-ai-ai/spec.md`

## 概述

本项目旨在开发一款嵌入式AI语音交互盒子,用于将毛绒玩具转变为可对话的智能玩具。系统由三部分组成:
1. **ESP32S3固件**: 负责语音采集、动作识别、音频播放、网络管理
2. **云端AI服务**: 提供语音识别(ASR)、AI对话生成、语音合成(TTS)
3. **配套移动APP**: 支持WiFi配网、设备管理(微信小程序或原生APP)

技术路径:使用ESP-IDF框架开发嵌入式固件,通过WiFi连接云端AI服务,采用零存储策略确保用户隐私保护,支持多WiFi网络管理和自动切换。

## 技术上下文

**语言/版本**:
- 固件: C/C++ (ESP-IDF v5.1+)
- 云端服务: Python 3.11+ (FastAPI/Flask) 或 Node.js 18+ (Express) [NEEDS CLARIFICATION]
- 移动APP: 微信小程序(WXML/JavaScript) 或 React Native [NEEDS CLARIFICATION]

**主要依赖**:
- 固件: ESP-IDF, FreeRTOS, mbedTLS, NVS, WiFi Manager
- 云端: AI服务SDK (OpenAI/Azure/阿里云通义千问) [NEEDS CLARIFICATION], WebSocket/HTTP服务器
- APP: 蓝牙通信库, 微信小程序API 或 React Native蓝牙模块

**存储**:
- 固件: ESP32S3 Flash (4-8MB), NVS分区存储WiFi凭证(加密)
- 云端: 无状态设计,不持久化用户数据(符合隐私要求)
- APP: 本地存储设备配对信息

**测试**:
- 固件: Unity测试框架(ESP-IDF集成), 硬件在环测试(HIL)
- 云端: pytest (Python) 或 Jest (Node.js)
- APP: 微信开发者工具 或 React Native Testing Library
- 集成: 端到端测试(固件↔云端↔APP)

**目标平台**:
- 固件: ESP32S3 (Xtensa LX7双核, 240MHz, 512KB SRAM)
- 云端: Linux服务器 (Docker容器化部署)
- APP: 微信小程序(微信7.0+) 或 iOS 13+/Android 8+

**项目类型**: 嵌入式IoT系统 + 云端服务 + 移动APP (三端架构)

**性能目标**:
- 语音响应延迟: ≤2秒(用户说话结束到AI回应开始播放)
- 打断检测响应: ≤500毫秒
- WiFi连接成功率: ≥95% (信号≥-70dBm)
- WiFi自动切换: ≤30秒
- 休眠唤醒: ≤3秒
- IMU动作识别准确率: ≥85%

**约束**:
- 固件Flash空间: ≤8MB (包含固件、OTA分区、系统提示音)
- 运行时RAM: ≤256KB (FreeRTOS + 应用)
- 续航时间: ≥6小时 (中等使用强度)
- 休眠功耗: ≤待机功耗的10%
- 充电温度保护: 45°C停止充电, 40°C恢复
- 零用户数据存储: 语音数据实时处理后立即丢弃
- WiFi凭证存储上限: ≥5个网络

**规模/范围**:
- 固件代码规模: 约10,000-15,000行C/C++
- 云端服务: 约2,000-3,000行Python/Node.js
- 移动APP: 约1,500-2,500行(小程序) 或 3,000-5,000行(原生)
- 系统提示音: 约13个音频文件(MP3/WAV, 总计<1MB)
- 支持用户规模: 初期1,000-10,000台设备

## Constitution检查

*说明: 项目constitution文件尚未定义(使用模板)。以下为基于嵌入式IoT最佳实践的基础检查项。*

### 基础质量门禁

| 检查项 | 状态 | 说明 |
| -------- | -------- | -------- |
| 明确的模块边界 | ✅ PASS | 固件、云端、APP三端分离,职责清晰 |
| 测试策略定义 | ✅ PASS | Unity(固件)、pytest/Jest(云端)、端到端测试 |
| 隐私合规 | ✅ PASS | 零存储策略,符合个人信息保护要求 |
| 错误处理 | ✅ PASS | API重试、温度保护、网络故障恢复策略明确 |
| 文档要求 | ✅ PASS | 包含quickstart.md、contracts/、data-model.md |

**结论**: 通过Phase 0研究门禁,可进入研究阶段。

## 项目结构

### 文档结构 (当前功能)

```
specs/001-ai-ai-ai/
├── spec.md              # 功能规格说明
├── plan.md              # 本文件 - 实现计划
├── research.md          # Phase 0输出 - 技术决策研究
├── data-model.md        # Phase 1输出 - 数据模型设计
├── quickstart.md        # Phase 1输出 - 快速开始指南
├── contracts/           # Phase 1输出 - API合约定义
│   ├── firmware-cloud-api.md      # 固件↔云端通信协议
│   ├── app-firmware-ble.md        # APP↔固件蓝牙协议
│   └── cloud-ai-service-api.md    # 云端↔AI服务接口
├── checklists/
│   └── requirements.md  # 规格质量检查清单
└── tasks.md             # Phase 2输出 - 任务分解 (由/speckit.tasks生成)
```

### 源代码结构 (仓库根目录)

```
# 选项: 嵌入式IoT三端架构

firmware/                      # ESP32S3固件
├── main/
│   ├── app_main.c            # 主入口
│   ├── audio/                # 音频子系统
│   │   ├── audio_player.c    # 扬声器播放
│   │   ├── audio_recorder.c  # 麦克风录音
│   │   └── audio_codec.c     # 音频编解码
│   ├── network/              # 网络子系统
│   │   ├── wifi_manager.c    # WiFi管理(多网络)
│   │   ├── ble_server.c      # 蓝牙配网服务
│   │   └── http_client.c     # HTTP/WebSocket客户端
│   ├── sensors/              # 传感器子系统
│   │   ├── imu_driver.c      # IMU驱动
│   │   ├── battery_monitor.c # 电池监测
│   │   └── temp_sensor.c     # 温度传感器
│   ├── ai/                   # AI交互子系统
│   │   ├── voice_session.c   # 语音会话管理
│   │   ├── gesture_detect.c  # 动作识别
│   │   └── interrupt_detect.c# 打断检测
│   ├── power/                # 电源管理
│   │   ├── power_manager.c   # 电源状态机
│   │   ├── sleep_manager.c   # 休眠/唤醒
│   │   └── charge_control.c  # 充电管理
│   ├── storage/              # 存储管理
│   │   └── nvs_config.c      # NVS配置存储
│   └── utils/
│       ├── system_audio.c    # 系统提示音
│       └── button_handler.c  # 按钮处理
├── components/               # 自定义组件
│   ├── audio_prompts/        # 系统提示音资源(嵌入)
│   └── wifi_multi/           # 多WiFi管理库
├── test/                     # Unity单元测试
└── CMakeLists.txt

cloud/                         # 云端AI服务
├── src/ 或 app/
│   ├── api/                  # API端点
│   │   ├── voice.py/js       # 语音处理API
│   │   └── health.py/js      # 健康检查
│   ├── services/             # 业务逻辑
│   │   ├── asr_service.py/js # 语音识别集成
│   │   ├── ai_service.py/js  # AI对话集成
│   │   └── tts_service.py/js # 语音合成集成
│   ├── models/               # 数据模型(请求/响应)
│   └── middleware/           # 中间件(日志、监控)
├── tests/                    # pytest或Jest测试
├── Dockerfile
└── requirements.txt 或 package.json

app/                          # 移动APP (微信小程序示例)
├── pages/
│   ├── index/                # 设备列表页
│   ├── config/               # WiFi配网页
│   └── manage/               # 设备管理页
├── components/
│   ├── ble-connector/        # 蓝牙连接组件
│   └── wifi-list/            # WiFi列表组件
├── utils/
│   ├── ble.js                # 蓝牙通信封装
│   └── protocol.js           # 通信协议编解码
├── app.json                  # 小程序配置
└── tests/

共享/
├── docs/                     # 共享文档
│   ├── protocol.md           # 通信协议文档
│   └── architecture.md       # 系统架构文档
└── tools/                    # 开发工具
    ├── audio_converter.py    # 音频格式转换工具
    └── firmware_flasher.sh   # 固件烧录脚本
```

**结构决策**:
采用三端分离架构,每个端独立开发和测试:
- **firmware/**: ESP-IDF标准项目结构,按功能模块组织(audio/network/sensors/ai/power)
- **cloud/**: 标准Web服务结构,采用分层架构(API层、服务层、模型层)
- **app/**: 微信小程序标准结构(pages/components/utils)或React Native结构

选择理由:
1. 三端独立开发,降低耦合,便于并行开发
2. 固件按硬件子系统组织,符合嵌入式开发习惯
3. 云端采用成熟的Web服务架构,便于扩展
4. 通过contracts/明确接口定义,确保三端协同

## 复杂度跟踪

*说明: Constitution检查全部通过,无需复杂度豁免*

| 违规项 | 为何需要 | 为何拒绝更简方案 |
| -------- | -------- | -------- |
| 无 | - | - |

## Phase 0: 研究大纲

### 需要澄清的技术决策

以下技术选型标记为"NEEDS CLARIFICATION",需通过研究确定:

1. **云端服务技术栈**: Python (FastAPI/Flask) vs Node.js (Express)
   - 研究重点: AI服务SDK支持、性能、开发效率、运维成本

2. **移动APP技术选型**: 微信小程序 vs React Native原生APP
   - 研究重点: 蓝牙API支持、开发周期、用户覆盖率、维护成本

3. **AI服务提供商**: OpenAI vs Azure Cognitive Services vs 阿里云通义千问 vs 讯飞星火
   - 研究重点: 中文识别准确率、语音识别质量、延迟、成本、合规性

4. **音频编解码**: Opus vs AAC vs ADPCM
   - 研究重点: 压缩率、ESP32S3硬件支持、延迟、带宽消耗

5. **通信协议**: WebSocket vs HTTP/2 vs MQTT
   - 研究重点: 双向通信、打断检测实时性、资源消耗、云端支持

6. **WiFi管理策略**: 自研 vs ESP-IDF WiFi Manager组件
   - 研究重点: 多WiFi支持、自动切换可靠性、Flash占用

7. **OTA升级方案**: ESP-IDF原生OTA vs 自定义方案
   - 研究重点: 回滚机制、安全性、Flash分区设计

### 研究任务列表

将在research.md中产出以下决策文档:

1. 云端技术栈选型
2. 移动APP开发框架选型
3. AI服务提供商评估
4. 音频传输方案设计
5. 通信协议架构
6. 多WiFi管理方案
7. 固件OTA策略 (标记为未来增强,Phase 1不实现)

**Phase 0输出**: `research.md` - 包含所有技术决策、选型理由、替代方案对比

---

*Phase 1和Phase 2将在Phase 0完成后继续*
