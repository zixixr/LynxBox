# API合约: 云端服务 ↔ AI服务提供商

**版本**: v1.0 | **提供商**: 讯飞 + 阿里云通义千问 | **日期**: 2025-10-07

## 概述

定义LynxBox云端服务与第三方AI服务(ASR、对话、TTS)的集成接口。

---

## 1. ASR服务 - 讯飞实时语音转写

### 1.1 WebSocket连接

**端点**: `wss://rtasr.xfyun.cn/v1/ws`

**认证**: URL签名(HMAC-SHA256)

**请求头**:
```
Authorization: Bearer <签名>
X-Appid: <应用ID>
X-CurTime: <当前时间戳>
```

### 1.2 音频参数配置

**首帧发送**(JSON):
```json
{
  "common": {
    "app_id": "5f7xxxxx"
  },
  "business": {
    "language": "zh_cn",
    "domain": "iat",
    "accent": "mandarin",
    "vad_eos": 1000,
    "dwa": "wpgs",
    "pd": "children"
  },
  "data": {
    "status": 0,
    "format": "audio/L16;rate=16000",
    "encoding": "raw",
    "audio": ""
  }
}
```

**参数说明**:
- `pd`: "children" - 儿童语音模型
- `vad_eos`: 1000ms - 静音检测阈值
- `dwa`: "wpgs" - 启用动态修正

### 1.3 音频数据传输

**后续帧**(二进制):
```json
{
  "data": {
    "status": 1,
    "format": "audio/L16;rate=16000",
    "encoding": "raw",
    "audio": "<Base64编码的PCM数据>"
  }
}
```

**最后一帧**:
```json
{
  "data": {
    "status": 2,
    "format": "audio/L16;rate=16000",
    "encoding": "raw",
    "audio": ""
  }
}
```

### 1.4 识别结果

**实时结果**(JSON):
```json
{
  "code": "0",
  "message": "success",
  "sid": "xxx@xxx",
  "data": {
    "result": {
      "sn": 1,
      "ls": false,
      "bg": 0,
      "ed": 0,
      "ws": [
        {
          "bg": 0,
          "cw": [
            {
              "w": "你好",
              "wp": "n",
              "wc": 0.98
            }
          ]
        }
      ]
    }
  }
}
```

**字段说明**:
- `ls`: false=中间结果, true=最终结果
- `wc`: 置信度 (0-1)
- `w`: 识别文本

### 1.5 错误处理

**错误码**:
- `10105`: 非法参数
- `10106`: 无权限
- `10114`: 音频格式错误
- `10160`: 请求超时

---

## 2. AI对话服务 - 阿里云通义千问

### 2.1 HTTP API

**端点**: `https://dashscope.aliyuncs.com/api/v1/services/aigc/text-generation/generation`

**方法**: POST

**认证**: HTTP头
```
Authorization: Bearer <API-KEY>
Content-Type: application/json
```

### 2.2 请求格式

```json
{
  "model": "qwen-turbo",
  "input": {
    "messages": [
      {
        "role": "system",
        "content": "你是一只可爱的小熊玩具,名叫Lynx,喜欢和小朋友玩耍。"
      },
      {
        "role": "user",
        "content": "你好"
      }
    ]
  },
  "parameters": {
    "temperature": 0.8,
    "top_p": 0.9,
    "max_tokens": 150,
    "result_format": "message"
  }
}
```

**参数说明**:
- `temperature`: 0.8 - 创造性(0.7-0.9适合对话)
- `max_tokens`: 150 - 最大输出长度(控制响应简洁)
- `result_format`: "message" - 返回完整消息格式

### 2.3 响应格式

```json
{
  "output": {
    "text": null,
    "finish_reason": "stop",
    "choices": [
      {
        "finish_reason": "stop",
        "message": {
          "role": "assistant",
          "content": "你好呀,小朋友!我是Lynx,很高兴见到你!"
        }
      }
    ]
  },
  "usage": {
    "total_tokens": 45,
    "input_tokens": 30,
    "output_tokens": 15
  },
  "request_id": "xxx-xxx-xxx"
}
```

### 2.4 上下文管理

**单轮对话策略**:
- 仅传递system prompt + 当前用户输入
- 不保存历史对话(符合零存储要求)
- 如需要上下文,通过gesture传递

**带动作上下文示例**:
```json
{
  "model": "qwen-turbo",
  "input": {
    "messages": [
      {
        "role": "system",
        "content": "你是小熊Lynx。用户刚刚摇晃了你,在回复时提及这个动作。"
      },
      {
        "role": "user",
        "content": "你好"
      }
    ]
  }
}
```

### 2.5 错误处理

**错误码**:
- `InvalidParameter`: 参数错误
- `AccessDenied`: 无权限
- `Throttling.RateQuota`: 超出配额
- `InternalError.Algo`: AI服务内部错误

---

## 3. TTS服务 - 讯飞语音合成

### 3.1 WebSocket连接

**端点**: `wss://tts-api.xfyun.cn/v2/tts`

**认证**: URL签名

### 3.2 请求格式

**首帧**(JSON):
```json
{
  "common": {
    "app_id": "5f7xxxxx"
  },
  "business": {
    "aue": "lame",
    "auf": "audio/L16;rate=16000",
    "vcn": "xiaoyan",
    "speed": 50,
    "volume": 50,
    "pitch": 50,
    "bgs": 0,
    "tte": "UTF8"
  },
  "data": {
    "status": 2,
    "text": "<Base64编码的UTF-8文本>"
  }
}
```

**参数说明**:
- `aue`: "lame" - MP3格式输出
- `vcn`: "xiaoyan" - 讯飞小燕(儿童女声)
- `speed`: 50 - 语速(0-100, 50=中速)
- `volume`: 50 - 音量
- `tte`: "UTF8" - 文本编码

### 3.3 响应格式

**音频数据帧**(JSON):
```json
{
  "code": 0,
  "message": "success",
  "sid": "xxx@xxx",
  "data": {
    "audio": "<Base64编码的MP3数据>",
    "status": 1,
    "ced": "UTF8"
  }
}
```

**最后一帧**:
```json
{
  "code": 0,
  "message": "success",
  "sid": "xxx@xxx",
  "data": {
    "audio": "<Base64编码的MP3数据>",
    "status": 2,
    "ced": "UTF8"
  }
}
```

**字段说明**:
- `status`: 1=中间帧, 2=最后一帧
- `audio`: Base64编码的MP3音频切片

### 3.4 音频格式转换

**TTS输出**: MP3, 16kHz, 64kbps

**需转换为**: Opus, 16kHz, 32kbps (发送给固件)

**转换流程** (Python):
```python
import subprocess
from pydub import AudioSegment

def mp3_to_opus(mp3_data: bytes) -> bytes:
    # 使用pydub + ffmpeg转换
    audio = AudioSegment.from_mp3(BytesIO(mp3_data))
    opus_data = audio.export(format="opus", bitrate="32k").read()
    return opus_data
```

---

## 4. 完整对话流程

```
1. 固件 → 云端: WebSocket session_start + Opus音频流

2. 云端 → 讯飞ASR:
   - 解码Opus → PCM 16kHz
   - 通过WebSocket发送实时语音转写请求

3. 讯飞ASR → 云端: 实时返回识别文本

4. 云端 → 通义千问:
   - 拼接system prompt + 用户输入 + gesture上下文
   - HTTP POST请求AI对话

5. 通义千问 → 云端: 返回AI文本回应

6. 云端 → 讯飞TTS:
   - 发送AI文本合成请求
   - 接收MP3音频流

7. 云端处理:
   - MP3解码 → Opus编码
   - 分帧(20ms/帧)

8. 云端 → 固件: WebSocket发送Opus音频帧流

9. 固件: 实时解码播放
```

---

## 5. 性能优化

### 5.1 并行处理

- ASR与AI请求可部分并行(ASR识别前几个字时提前发起AI请求)
- TTS合成采用流式输出,边合成边发送

### 5.2 缓存策略

- 常见问候语TTS缓存(如"你好" → 预合成Opus)
- 减少TTS调用延迟

### 5.3 超时控制

| 服务 | 超时时间 | 重试策略 |
| -------- | -------- | -------- |
| 讯飞ASR | 5秒 | 不重试(实时服务) |
| 通义千问 | 5秒 | 重试1次 (FR-015b) |
| 讯飞TTS | 5秒 | 重试1次 |

---

## 6. 成本预估

### 6.1 单次对话成本

| 服务 | 调用次数 | 单价 | 成本 |
| -------- | -------- | -------- | -------- |
| 讯飞ASR | 1次 | ¥0.003/次 | ¥0.003 |
| 通义千问 | 1次 | ¥0.002/千token (平均50token) | ¥0.0001 |
| 讯飞TTS | 1次 | ¥0.002/次 | ¥0.002 |
| **总计** | - | - | **¥0.0051** |

### 6.2 月度成本预估

假设1000台设备,每台每天对话10次:
- 日对话次数: 1000 × 10 = 10,000
- 月对话次数: 10,000 × 30 = 300,000
- **月度成本**: 300,000 × ¥0.0051 ≈ **¥1,530**

---

**下一步**: 实现云端服务集成这三个AI服务API
