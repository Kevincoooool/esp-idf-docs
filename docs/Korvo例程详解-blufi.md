# Korvo · BluFi 蓝牙配网

> **适用开发板**：Korvo S3 · **工程** `02.beginner.blufi` · **target** `esp32s3`  
> **通用概念**：[配网教程](./通用-配网.md)  
> **能力**：**BluFi** — 手机 App 经蓝牙把 Wi-Fi SSID/密码下发给设备  

产品常见「无屏配网」方案：用户未在 menuconfig 写死 Wi-Fi 时，用 Espressif **BluFi** 协议完成安全协商与配置。与单纯 BLE 串口透传不同，是一套完整 **配网状态机**。

---

## 1. 这个例子在干嘛

```text
上电 → 进入 BluFi 广播/连接
  → 手机 EspBlufi App 连接
  → 安全协商 (blufi_security.c)
  → 收到 SSID/密码 (回调)
  → 发起 Wi-Fi 连接
  → 屏幕 + 串口显示配网/联网状态
```

Wi-Fi 状态与 BLE 状态 **并行**，需在回调里理清何时 `esp_wifi_connect`。

---

## 2. 编译烧录

```powershell
cd Korvo_Firmware\02.beginner.blufi
idf.py set-target esp32s3
idf.py build flash monitor
```

---

## 3. 手机端

安装 Espressif **EspBlufi**（或文档指定 App）：

1. 打开蓝牙，搜索板子 BluFi 设备名（可在源码改广播名）  
2. 连接后按 App 流程发送当前手机 Wi-Fi 信息或手动输入  
3. 配网成功则板子连接路由器，log 出现 `GOT IP`  

---

## 4. 关键文件

| 文件 | 作用 |
|------|------|
| `main/app_main.c` | 主流程、状态展示 |
| `main/blufi_init.c` | BluFi 初始化、回调注册 |
| `main/blufi_security.c` | DH/加密协商 |

---

## 5. 重点理解

- **BluFi 不是 SPP**：协议与 GATT 服务由 ESP-IDF BluFi 组件定义  
- **安全**：`blufi_security.c` 处理密钥，勿随意关验证  
- **配网完成后**：通常停止 BluFi 广播或进入待机（以实现为准）  

---

## 6. 常见问题

| 现象 | 处理 |
|------|------|
| App 搜不到 | 蓝牙天线；设备名；是否已在广播 |
| 配网失败 | 2.4G 密码；路由器隐藏 SSID |
| 连上又断 | 回调里重复 connect；看 Wi-Fi 事件 log |

---

## 7. 动手改造

1. 改 BluFi 广播设备名  
2. 配网成功后在屏幕大字提示 IP  
3. 配网成功后自动连接 MQTT  

---

## 8. 下一步

[wifi_station](./Korvo例程详解-wifi_station.md) · `02.beginner.ble_spp_server`

酷世DIY · Kevincoooool
