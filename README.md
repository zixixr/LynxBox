# LynxBox - AI语音交互盒子 🎙️🧸

> 将普通毛绒玩具变成可以对话交流的智能AI玩具

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Platform: ESP32S3](https://img.shields.io/badge/Platform-ESP32S3-blue.svg)](https://www.espressif.com/en/products/socs/esp32-s3)
[![Python: 3.11+](https://img.shields.io/badge/Python-3.11+-green.svg)](https://www.python.org/)
[![Framework: FastAPI](https://img.shields.io/badge/Framework-FastAPI-009688.svg)](https://fastapi.tiangolo.com/)

## 📖 项目简介

LynxBox是一款嵌入式AI语音交互设备,专为将毛绒玩具转变为智能对话玩具而设计。通过集成先进的语音识别、AI对话和语音合成技术,让用户可以与心爱的玩具进行自然的语音交互。

### ✨ 核心特性

- 🎤 **实时语音对话** - 支持自然语音交互,端到端延迟<2秒
- 🛑 **智能打断检测** - 双麦克风阵列,500ms内响应用户打断
- 🏃 **动作识别** - 通过IMU传感器识别摇晃/翻转/轻拍等动作
- 📡 **多WiFi管理** - 支持保存5个网络,自动切换最强信号
- 🔐 **隐私保护** - 零用户数据存储,符合隐私保护法规
- 🔋 **智能电源** - 10分钟无操作自动休眠,续航≥6小时
- 🌡️ **安全充电** - 温度保护机制,充电期间禁用交互
- 📱 **便捷配网** - 微信小程序一键WiFi配置

## 🏗️ 系统架构

```
┌─────────────────┐       WebSocket/TLS        ┌─────────────────┐
│   ESP32S3固件    │ ◄──────────────────────► │   云端服务      │
│                 │    Opus音频流(32kbps)     │   FastAPI       │
│  • 音频采集/播放 │    控制消息(JSON)         │   WebSocket     │
│  • WiFi管理     │                           │                 │
│  • 动作识别     │                           │  • 讯飞ASR      │
│  • 电源管理     │                           │  • 通义千问AI   │
└─────────────────┘                           │  • 讯飞TTS      │
        ▲                                     └─────────────────┘
        │ BLE GATT                                    ▲
        │ WiFi配网                                    │ AI服务API
        ▼                                             │
┌─────────────────┐                                   │
│  微信小程序      │                                   │
│                 │                                   ▼
│  • 设备扫描     │                           ┌─────────────────┐
│  • WiFi配置     │                           │ 讯飞开放平台 +   │
│  • 网络管理     │                           │ 阿里云通义千问   │
└─────────────────┘                           └─────────────────┘
```

## 🚀 快速开始

### 前置要求

**硬件:**
- ESP32-S3开发板
- 双麦克风模块(I2S,如INMP441)
- 扬声器/功放(I2S DAC,如MAX98357A)
- IMU传感器(I2C,如MPU6050)
- USB-C线

**软件:**
- [ESP-IDF v5.1+](https://docs.espressif.com/projects/esp-idf/)
- Python 3.11+
- 微信开发者工具
- Git

### 15分钟快速搭建

详细步骤请参考: [📘 快速开始指南](specs/001-ai-ai-ai/quickstart.md)

#### 1. 克隆仓库

```bash
git clone https://github.com/zixixr/LynxBox.git
cd LynxBox
```

#### 2. 固件开发环境

```bash
# 安装ESP-IDF (参考官方文档)
# macOS/Linux:
cd firmware
idf.py set-target esp32s3
idf.py build
idf.py -p /dev/ttyUSB0 flash monitor
```

#### 3. 云端服务

```bash
cd cloud
python3 -m venv venv
source venv/bin/activate  # Windows: venv\Scripts\activate
pip install -r requirements.txt
python main.py
```

#### 4. 微信小程序

```bash
# 使用微信开发者工具打开 app/ 目录
# 配置AppID并编译运行
```

## 📚 文档

### 核心文档

| 文档 | 描述 |
|------|------|
| [功能规格说明](specs/001-ai-ai-ai/spec.md) | 完整的需求定义和用户场景 |
| [实现计划](specs/001-ai-ai-ai/plan.md) | 技术选型和系统架构设计 |
| [技术研究](specs/001-ai-ai-ai/research.md) | 7个关键技术决策及理由 |
| [数据模型](specs/001-ai-ai-ai/data-model.md) | 6个核心实体设计 |
| [快速入门](specs/001-ai-ai-ai/quickstart.md) | 15分钟开发环境搭建 |

### API合约

| 合约 | 描述 |
|------|------|
| [固件↔云端](specs/001-ai-ai-ai/contracts/firmware-cloud-api.md) | WebSocket音频流 + HTTP API |
| [APP↔固件](specs/001-ai-ai-ai/contracts/app-firmware-ble.md) | 蓝牙BLE配网协议 |
| [云端↔AI](specs/001-ai-ai-ai/contracts/cloud-ai-service-api.md) | 讯飞+通义千问集成 |

## 🛠️ 技术栈

### 固件 (ESP32S3)

- **框架**: ESP-IDF v5.1
- **语言**: C/C++
- **音频编解码**: Opus (32kbps)
- **网络**: WiFi (2.4GHz) + BLE 4.2
- **存储**: NVS (加密WiFi凭证)

### 云端服务

- **语言**: Python 3.11
- **框架**: FastAPI
- **ASR**: 讯飞实时语音转写
- **AI对话**: 阿里云通义千问Qwen-Turbo
- **TTS**: 讯飞语音合成
- **部署**: Docker + Uvicorn

### 移动APP

- **平台**: 微信小程序
- **语言**: JavaScript (ES6+)
- **蓝牙**: 小程序BLE API
- **UI**: WeUI组件库

## 📊 性能指标

| 指标 | 目标值 | 实际值 |
|------|--------|--------|
| 语音响应延迟 | ≤2秒 | 🚧 开发中 |
| 打断检测响应 | ≤500ms | 🚧 开发中 |
| WiFi连接成功率 | ≥95% | 🚧 开发中 |
| WiFi自动切换 | ≤30秒 | 🚧 开发中 |
| 续航时间 | ≥6小时 | 🚧 开发中 |
| IMU识别准确率 | ≥85% | 🚧 开发中 |

## 🔐 隐私与安全

- ✅ **零存储策略**: 语音数据实时处理后立即丢弃,云端不保存任何对话记录
- ✅ **加密传输**: TLS/SSL加密所有网络通信
- ✅ **WiFi密码保护**: AES-256加密存储于设备本地
- ✅ **合规认证**: 符合《个人信息保护法》等隐私保护规定
- ✅ **数据本地化**: 使用国内AI服务,数据不出境

## 📈 项目状态

- [x] Phase 0: 技术研究与决策 ✅
- [x] Phase 1: 系统设计与API合约 ✅
- [ ] Phase 2: 任务分解 🚧
- [ ] Phase 3: 固件开发 📅 计划中
- [ ] Phase 4: 云端服务开发 📅 计划中
- [ ] Phase 5: 移动APP开发 📅 计划中
- [ ] Phase 6: 集成测试 📅 计划中
- [ ] Phase 7: 硬件适配 📅 计划中

## 🤝 贡献指南

欢迎贡献代码、文档或提出问题!

1. Fork本仓库
2. 创建特性分支 (`git checkout -b feature/AmazingFeature`)
3. 提交更改 (`git commit -m 'Add some AmazingFeature'`)
4. 推送到分支 (`git push origin feature/AmazingFeature`)
5. 开启Pull Request

### 开发规范

- 固件代码遵循ESP-IDF编码规范
- Python代码遵循PEP 8
- 提交信息遵循[Conventional Commits](https://www.conventionalcommits.org/)
- 所有API变更需更新contracts文档

## 📝 许可证

本项目采用 [MIT License](LICENSE) 许可证。

## 🙏 致谢

- [ESP-IDF](https://github.com/espressif/esp-idf) - Espressif物联网开发框架
- [FastAPI](https://fastapi.tiangolo.com/) - 现代Python Web框架
- [讯飞开放平台](https://www.xfyun.cn/) - 语音识别与合成服务
- [阿里云百炼](https://www.aliyun.com/product/bailian) - 通义千问AI服务

## 📧 联系方式

- **Issues**: [GitHub Issues](https://github.com/zixixr/LynxBox/issues)
- **讨论**: [GitHub Discussions](https://github.com/zixixr/LynxBox/discussions)

---

<p align="center">
  用❤️和🤖构建 | Made with ❤️ and 🤖
</p>

<p align="center">
  <sub>🤖 Generated with <a href="https://claude.com/claude-code">Claude Code</a></sub>
</p>
