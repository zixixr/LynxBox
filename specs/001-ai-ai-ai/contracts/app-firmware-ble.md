# API合约: 移动APP ↔ 固件 (蓝牙BLE)

**版本**: v1.0 | **协议**: BLE GATT | **日期**: 2025-10-07

## 概述

定义微信小程序/原生APP与ESP32S3固件之间的蓝牙低功耗(BLE)通信协议,用于WiFi配网和设备管理。

---

## 1. BLE服务定义

### 1.1 主服务 (Primary Service)

**UUID**: `0000FF00-0000-1000-8000-00805F9B34FB` (自定义)

**服务名称**: LynxBox Configuration Service

---

### 1.2 特征值 (Characteristics)

#### 特征1: WiFi配置写入

**UUID**: `0000FF01-0000-1000-8000-00805F9B34FB`

**属性**: Write, Write Without Response

**用途**: APP写入WiFi凭证

**数据格式** (JSON字符串,最大512字节):
```json
{
  "cmd": "add_wifi",
  "ssid": "HomeWiFi",
  "password": "12345678"
}
```

**响应**: 通过特征2通知结果

---

#### 特征2: 配置状态通知

**UUID**: `0000FF02-0000-1000-8000-00805F9B34FB`

**属性**: Notify, Read

**用途**: 固件通知APP配网结果

**数据格式** (JSON字符串):
```json
{
  "status": "success",
  "ssid": "HomeWiFi",
  "rssi": -65,
  "timestamp": 1234567890
}
```

**状态码**:
- `success`: 连接成功
- `failed`: 连接失败
- `timeout`: 连接超时(30秒)
- `invalid_password`: 密码错误

---

#### 特征3: 设备信息读取

**UUID**: `0000FF03-0000-1000-8000-00805F9B34FB`

**属性**: Read

**用途**: APP读取设备基本信息

**数据格式** (JSON字符串):
```json
{
  "device_id": "lynx_abc12345_def678",
  "mac": "AA:BB:CC:DD:EE:FF",
  "firmware_version": "v1.0.0",
  "battery_level": 85,
  "saved_networks": 3
}
```

---

#### 特征4: WiFi列表管理

**UUID**: `0000FF04-0000-1000-8000-00805F9B34FB`

**属性**: Read, Write

**用途**: 读取已保存WiFi列表,删除指定网络

**读取时返回** (JSON数组):
```json
{
  "cmd": "list_wifi",
  "networks": [
    {
      "index": 0,
      "ssid": "HomeWiFi",
      "rssi": -65,
      "last_connect": 1234567890
    },
    {
      "index": 1,
      "ssid": "OfficeWiFi",
      "rssi": -70,
      "last_connect": 1234567800
    }
  ]
}
```

**删除网络时写入**:
```json
{
  "cmd": "delete_wifi",
  "index": 1
}
```

**响应**: 通过特征2通知结果

---

## 2. 配网流程

### 2.1 完整流程

```
1. 盒子开机进入蓝牙配对模式
   - 播放提示音"请使用手机APP连接我"
   - 开始BLE广播 (名称: LynxBox-XXXX)

2. APP扫描BLE设备
   wx.openBluetoothAdapter()
   wx.startBluetoothDevicesDiscovery({ services: ['0000FF00-...'] })

3. APP连接设备
   wx.createBLEConnection({ deviceId })

4. APP发现服务和特征
   wx.getBLEDeviceServices({ deviceId })
   wx.getBLEDeviceCharacteristics({ deviceId, serviceId })

5. APP订阅状态通知
   wx.notifyBLECharacteristicValueChange({
     characteristicId: '0000FF02-...',
     state: true
   })

6. APP读取设备信息
   wx.readBLECharacteristicValue({ characteristicId: '0000FF03-...' })

7. APP写入WiFi凭证
   wx.writeBLECharacteristicValue({
     characteristicId: '0000FF01-...',
     value: ArrayBuffer (JSON字符串转码)
   })

8. 固件尝试连接WiFi (30秒超时)

9. 固件通知APP结果
   通过特征0000FF02发送通知

10. 连接成功后固件关闭BLE,APP断开连接
```

---

## 3. 命令定义

### 3.1 APP → 固件命令

#### 添加WiFi网络

**特征**: 0000FF01

**命令**:
```json
{
  "cmd": "add_wifi",
  "ssid": "HomeWiFi",
  "password": "12345678"
}
```

**验证规则**:
- SSID: 1-32字符,UTF-8
- 密码: 0字符(开放网络) 或 8-64字符(WPA2)

#### 删除WiFi网络

**特征**: 0000FF04

**命令**:
```json
{
  "cmd": "delete_wifi",
  "index": 1
}
```

**验证规则**:
- index: 0-4 (最多5个网络)

#### 查询WiFi列表

**特征**: 0000FF04

**命令**:
```json
{
  "cmd": "list_wifi"
}
```

#### 设备重置

**特征**: 0000FF01

**命令**:
```json
{
  "cmd": "factory_reset",
  "confirm": true
}
```

**效果**: 清除所有WiFi凭证,恢复出厂设置,自动关机

---

### 3.2 固件 → APP通知

#### 配网成功

**特征**: 0000FF02 (Notify)

```json
{
  "status": "success",
  "ssid": "HomeWiFi",
  "rssi": -65,
  "ip": "192.168.1.100",
  "timestamp": 1234567890
}
```

#### 配网失败

```json
{
  "status": "failed",
  "ssid": "HomeWiFi",
  "error_code": "invalid_password",
  "error_message": "密码错误",
  "timestamp": 1234567890
}
```

**错误码**:
- `invalid_password`: 密码错误
- `ssid_not_found`: SSID不存在
- `timeout`: 连接超时
- `dhcp_failed`: DHCP获取IP失败

#### 网络已满

```json
{
  "status": "error",
  "error_code": "network_limit",
  "error_message": "已保存5个网络,请先删除",
  "timestamp": 1234567890
}
```

---

## 4. 数据编码

### 4.1 JSON to ArrayBuffer (微信小程序)

```javascript
// APP侧编码
function stringToArrayBuffer(str) {
  const encoder = new TextEncoder();
  return encoder.encode(str).buffer;
}

// 发送WiFi凭证
const data = {
    cmd: "add_wifi",
    ssid: "HomeWiFi",
    password: "12345678"
  };
const buffer = stringToArrayBuffer(JSON.stringify(data));

wx.writeBLECharacteristicValue({
  deviceId: deviceId,
  serviceId: '0000FF00-0000-1000-8000-00805F9B34FB',
  characteristicId: '0000FF01-0000-1000-8000-00805F9B34FB',
  value: buffer
});
```

### 4.2 ArrayBuffer to JSON (固件侧)

```c
// 固件侧解码
void parse_ble_command(uint8_t *data, size_t len) {
    char json_str[512];
    memcpy(json_str, data, len);
    json_str[len] = '\0';

    cJSON *root = cJSON_Parse(json_str);
    const char *cmd = cJSON_GetObjectItem(root, "cmd")->valuestring;

    if (strcmp(cmd, "add_wifi") == 0) {
        const char *ssid = cJSON_GetObjectItem(root, "ssid")->valuestring;
        const char *password = cJSON_GetObjectItem(root, "password")->valuestring;
        // 处理添加WiFi...
    }

    cJSON_Delete(root);
}
```

---

## 5. 错误处理

### 5.1 BLE连接失败

- APP端: 提示用户"连接失败,请重试"
- 固件端: 保持BLE广播,等待下次连接

### 5.2 配网超时

- 固件30秒内未连接成功WiFi
- 播放提示音"网络连接失败,请重试"
- 保持BLE连接,等待APP重新发送凭证

### 5.3 JSON解析错误

- 固件端: 通过特征0000FF02通知APP
```json
{
  "status": "error",
  "error_code": "invalid_json",
  "error_message": "命令格式错误"
}
```

---

## 6. 安全性

### 6.1 蓝牙配对

- **无PIN配对**: 简化用户操作(Just Works模式)
- **连接加密**: BLE链路层加密
- **WiFi密码保护**: 传输后立即加密存储(AES-256),不在BLE链路明文传输

### 6.2 攻击防护

- **距离限制**: BLE有效距离<10米,降低远程攻击风险
- **超时机制**: 10分钟无操作自动关闭BLE (FR-023)
- **单连接**: 仅允许1个APP同时连接

---

## 7. 性能要求

| 指标 | 目标值 |
| -------- | -------- |
| BLE广播间隔 | 100ms |
| 连接建立时间 | <3秒 |
| 配网总时长 | <60秒(含WiFi连接30秒) |
| 数据传输速率 | ~20KB/s (BLE 4.2标准) |

---

**下一步**: 参考此合约实现微信小程序蓝牙模块和固件BLE Server
