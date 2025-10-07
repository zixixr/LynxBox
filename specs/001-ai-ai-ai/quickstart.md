# 开发快速入门: AI语音交互盒子

**项目**: LynxBox | **日期**: 2025-10-07 | **目标**: 15分钟搭建开发环境

## 概述

本指南帮助开发者快速搭建LynxBox三端(固件、云端、APP)开发环境,运行Hello World示例,并了解基本开发流程。

---

## 前置要求

### 硬件

- **ESP32S3开发板** (建议: ESP32-S3-DevKitC-1)
- **双麦克风模块** (I2S接口,如INMP441)
- **扬声器/功放** (I2S DAC,如MAX98357A)
- **IMU传感器** (I2C/SPI,如MPU6050)
- **USB线** (Type-C,用于烧录和调试)

### 软件

- **操作系统**: Windows 10/11, macOS 12+, Ubuntu 20.04+
- **ESP-IDF**: v5.1+
- **Python**: 3.11+
- **Node.js**: 18+ (如需运行云端)
- **微信开发者工具**: 最新版(如需开发小程序)
- **Git**: 2.30+

---

## Part 1: 固件开发环境 (ESP32S3)

### 1.1 安装ESP-IDF

#### Windows

```bash
# 下载ESP-IDF安装器
https://dl.espressif.com/dl/esp-idf/

# 运行安装器,选择ESP-IDF v5.1
# 安装路径: C:\Espressif\frameworks\esp-idf-v5.1

# 打开ESP-IDF PowerShell
# 验证安装
idf.py --version
```

#### macOS/Linux

```bash
# 安装依赖
# macOS:
brew install cmake ninja dfu-util

# Ubuntu:
sudo apt-get install git wget flex bison gperf python3 python3-pip \
    python3-venv cmake ninja-build ccache libffi-dev libssl-dev \
    dfu-util libusb-1.0-0

# 克隆ESP-IDF
mkdir -p ~/esp
cd ~/esp
git clone --recursive https://github.com/espressif/esp-idf.git
cd esp-idf
git checkout v5.1

# 安装工具链
./install.sh esp32s3

# 设置环境变量(添加到~/.bashrc或~/.zshrc)
alias get_idf='. $HOME/esp/esp-idf/export.sh'

# 激活环境
get_idf
```

### 1.2 创建Hello World固件项目

```bash
# 创建项目
cd ~/projects
idf.py create-project lynxbox-firmware
cd lynxbox-firmware

# 设置目标芯片
idf.py set-target esp32s3

# 配置项目
idf.py menuconfig
# 导航: Serial flasher config → Flash size → 设置为8MB

# 编译
idf.py build

# 连接ESP32S3开发板,烧录
idf.py -p COM3 flash monitor  # Windows
idf.py -p /dev/ttyUSB0 flash monitor  # Linux
idf.py -p /dev/cu.usbserial-* flash monitor  # macOS

# 按Ctrl+] 退出监视器
```

### 1.3 测试音频播放 (系统提示音)

创建 `main/test_audio.c`:

```c
#include "esp_log.h"
#include "driver/i2s_std.h"

static const char *TAG = "audio_test";

void app_main(void) {
    ESP_LOGI(TAG, "LynxBox Audio Test");

    // TODO: 配置I2S用于扬声器
    // TODO: 播放测试音频(440Hz正弦波)

    ESP_LOGI(TAG, "Audio playback complete");
}
```

---

## Part 2: 云端服务开发环境 (Python)

### 2.1 安装Python依赖

```bash
# 创建项目目录
mkdir lynxbox-cloud
cd lynxbox-cloud

# 创建虚拟环境
python3 -m venv venv

# 激活虚拟环境
# Windows:
venv\Scripts\activate
# macOS/Linux:
source venv/bin/activate

# 安装核心依赖
pip install fastapi uvicorn[standard] websockets python-multipart pydantic

# 安装AI服务SDK
pip install dashscope  # 阿里云通义千问
pip install websocket-client  # 讯飞WebSocket

# 安装音频处理库
pip install opuslib av pydub

# 保存依赖
pip freeze > requirements.txt
```

### 2.2 创建Hello World WebSocket服务器

创建 `main.py`:

```python
from fastapi import FastAPI, WebSocket
from fastapi.responses import JSONResponse
import uvicorn

app = FastAPI(title="LynxBox Cloud Service")

@app.get("/api/v1/health")
async def health_check():
    return JSONResponse({
        "status": "ok",
        "version": "v1.0.0",
        "services": {
            "asr": "online",
            "ai": "online",
            "tts": "online"
        }
    })

@app.websocket("/ws/v1/voice")
async def voice_websocket(websocket: WebSocket):
    await websocket.accept()
    print(f"Client connected: {websocket.client}")

    try:
        while True:
            data = await websocket.receive_text()
            print(f"Received: {data}")

            # Echo back
            await websocket.send_text(f"Echo: {data}")
    except Exception as e:
        print(f"Error: {e}")
    finally:
        await websocket.close()

if __name__ == "__main__":
    uvicorn.run(app, host="0.0.0.0", port=8000, log_level="info")
```

### 2.3 运行并测试

```bash
# 启动服务
python main.py

# 在另一个终端测试健康检查
curl http://localhost:8000/api/v1/health

# 测试WebSocket (使用wscat或浏览器)
npm install -g wscat
wscat -c ws://localhost:8000/ws/v1/voice
```

---

## Part 3: 移动APP开发环境 (微信小程序)

### 3.1 安装微信开发者工具

```bash
# 下载地址
https://developers.weixin.qq.com/miniprogram/dev/devtools/download.html

# 安装后登录微信账号

# 创建新项目
# AppID: 使用测试号或申请正式AppID
# 项目目录: ~/projects/lynxbox-app
# 模板: 不使用模板
```

### 3.2 创建Hello World小程序

#### 项目结构

```
lynxbox-app/
├── app.js          # 小程序逻辑
├── app.json        # 小程序配置
├── app.wxss        # 全局样式
├── pages/
│   └── index/
│       ├── index.js    # 页面逻辑
│       ├── index.json  # 页面配置
│       ├── index.wxml  # 页面结构
│       └── index.wxss  # 页面样式
└── utils/
    └── ble.js      # 蓝牙工具类
```

#### app.json

```json
{
  "pages": [
    "pages/index/index"
  ],
  "window": {
    "navigationBarTitleText": "LynxBox配网",
    "backgroundColor": "#F8F8F8"
  },
  "permission": {
    "scope.bluetooth": {
      "desc": "需要使用蓝牙连接设备"
    }
  }
}
```

#### pages/index/index.wxml

```xml
<view class="container">
  <text class="title">LynxBox 设备配网</text>

  <button type="primary" bindtap="startScan">
    扫描设备
  </button>

  <view class="device-list">
    <block wx:for="{{devices}}" wx:key="deviceId">
      <view class="device-item" bindtap="connectDevice" data-device-id="{{item.deviceId}}">
        <text>{{item.name}}</text>
        <text class="rssi">信号: {{item.RSSI}}</text>
      </view>
    </block>
  </view>
</view>
```

#### pages/index/index.js

```javascript
Page({
  data: {
    devices: []
  },

  onLoad() {
    console.log('LynxBox App Loaded');
  },

  startScan() {
    const that = this;

    // 初始化蓝牙
    wx.openBluetoothAdapter({
      success() {
        console.log('Bluetooth initialized');

        // 开始扫描
        wx.startBluetoothDevicesDiscovery({
          services: ['0000FF00-0000-1000-8000-00805F9B34FB'],
          success() {
            console.log('Scanning started');

            // 监听设备发现
            wx.onBluetoothDeviceFound((res) => {
              res.devices.forEach((device) => {
                if (device.name && device.name.startsWith('LynxBox')) {
                  console.log('Found device:', device);

                  that.setData({
                    devices: [...that.data.devices, device]
                  });
                }
              });
            });
          }
        });
      },
      fail(err) {
        console.error('Bluetooth init failed:', err);
        wx.showToast({ title: '请开启蓝牙', icon: 'none' });
      }
    });
  },

  connectDevice(e) {
    const deviceId = e.currentTarget.dataset.deviceId;
    console.log('Connecting to:', deviceId);

    wx.createBLEConnection({
      deviceId,
      success() {
        wx.showToast({ title: '连接成功', icon: 'success' });
        // TODO: 导航到WiFi配置页面
      },
      fail(err) {
        console.error('Connection failed:', err);
        wx.showToast({ title: '连接失败', icon: 'none' });
      }
    });
  }
});
```

### 3.3 测试小程序

1. 在微信开发者工具中打开项目
2. 点击"编译"运行小程序
3. 点击"扫描设备"按钮
4. 查看控制台日志确认蓝牙初始化成功

---

## Part 4: 端到端测试流程

### 4.1 准备测试环境

```
1. 云端服务: 运行在本地8000端口
   python main.py

2. 固件: 烧录到ESP32S3并上电
   idf.py -p COM3 flash monitor

3. 小程序: 在微信开发者工具中运行
   点击"编译"
```

### 4.2 测试配网流程

```
1. ESP32S3开机 → 播放"请使用手机APP连接我"
2. 小程序点击"扫描设备" → 发现LynxBox-XXXX
3. 点击设备 → 连接成功
4. 输入WiFi SSID和密码 → 发送到固件
5. 固件尝试连接WiFi → 30秒内成功
6. 播放"网络连接成功" → 关闭BLE
```

### 4.3 测试语音对话 (需要真实AI服务)

```
1. 固件连接云端WebSocket
   ws://localhost:8000/ws/v1/voice?device_id=lynx_test_123456

2. 用户对着麦克风说"你好"

3. 固件录音 → Opus编码 → WebSocket上传

4. 云端:
   - Opus解码 → PCM
   - 调用讯飞ASR识别
   - 调用通义千问生成回应
   - 调用讯飞TTS合成
   - Opus编码 → WebSocket下发

5. 固件接收Opus音频 → 解码播放
```

---

## Part 5: 常见问题

### Q1: ESP-IDF编译失败

```bash
# 清理构建
idf.py fullclean

# 重新配置
idf.py menuconfig

# 重新编译
idf.py build
```

### Q2: 固件烧录失败

```bash
# 按住BOOT按钮,按一下RST按钮,松开BOOT
# 然后执行烧录
idf.py -p COM3 flash
```

### Q3: 微信小程序蓝牙权限被拒绝

```
1. 微信开发者工具: 设置 → 项目设置 → 勾选"不校验合法域名"
2. 真机调试: 微信 → 我 → 设置 → 通用 → 发现页管理 → 小程序 → 开启蓝牙权限
```

### Q4: Python依赖安装失败

```bash
# 升级pip
pip install --upgrade pip setuptools wheel

# 使用国内镜像
pip install -i https://pypi.tuna.tsinghua.edu.cn/simple <package>
```

---

## Part 6: 下一步

完成快速入门后,建议阅读:

1. **contracts/**: API合约文档,了解三端通信协议
2. **data-model.md**: 数据模型设计,了解实体关系
3. **research.md**: 技术决策文档,了解选型理由

开始实现核心功能:
1. 固件: 实现音频录放、WiFi管理、WebSocket客户端
2. 云端: 集成讯飞ASR、通义千问、讯飞TTS
3. 小程序: 实现完整配网流程和设备管理

---

## 附录: 有用的资源

### 官方文档

- ESP-IDF编程指南: https://docs.espressif.com/projects/esp-idf/zh_CN/v5.1/esp32s3/
- FastAPI文档: https://fastapi.tiangolo.com/zh/
- 微信小程序文档: https://developers.weixin.qq.com/miniprogram/dev/framework/

### AI服务文档

- 讯飞开放平台: https://www.xfyun.cn/doc/
- 阿里云通义千问: https://help.aliyun.com/zh/dashscope/

### 社区与支持

- ESP32论坛: https://www.esp32.com/
- GitHub Issues: (项目仓库issues页面)

---

**恭喜!** 您已完成LynxBox开发环境搭建,可以开始核心功能开发了。
