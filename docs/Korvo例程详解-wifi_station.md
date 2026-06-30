# Korvo · Wi-Fi Station 联网

> **适用开发板**：ESP32-S3 Korvo 2 V3  
> **工程**：`Korvo_Firmware/02.beginner.wifi_station`  
> **target**：`esp32s3`  
> Wi-Fi 在 **S3 芯片内置**，上电后可直接连路由器，无需协处理器。  
> API 概念：[WiFi使用教程](./通用-wifi.md)

---

## 1. 这个例子在干嘛

Station 模式连接路由器，DHCP 获 IP；屏幕三行状态卡 + 串口同步显示 SSID、IP、重试次数。后续 HTTP/MQTT/OTA/网页图传都建在此之上。

---

## 2. 编译烧录

```powershell
cd Korvo_Firmware\02.beginner.wifi_station
idf.py set-target esp32s3
idf.py menuconfig    # Example Configuration → SSID/密码
idf.py build flash monitor
```

日常用 **左 USB（CH9102）** 下载与 log。

---

## 3. menuconfig

| 选项 | 说明 |
|------|------|
| `ESP_WIFI_SSID` / `ESP_WIFI_PASSWORD` | 路由器账号 |
| `ESP_MAXIMUM_RETRY` | 最大重连次数，默认 5 |

---

## 4. 关键文件

- `main/station_example_main.c` — 全部逻辑
- `components/ksdiy_example_display` — 屏幕 UI

---

## 5. 核心流程

```text
app_main → display_bootstrap → nvs → wifi_init_sta()
  → 事件：STA_START → esp_wifi_connect()
  → DISCONNECTED → 重连 + 屏幕显示 retry
  → GOT_IP → 显示 IP
```

Wi-Fi 是 **事件驱动**，不要写成阻塞 `while` 等连接。

---

## 6. 常见问题

- **一直 retry**：SSID/密码、是否 2.4G、路由器是否隐藏 SSID  
- **连上无 IP**：路由器 DHCP 池、MAC 过滤  
- **改 SSID 无效**：menuconfig 保存后重新 build flash  

---

## 7. 下一步（Korvo 仓库）

`02.beginner.wifi_scan` → [HTTP Server](./Korvo例程详解-http_server.md) → [MQTT](./Korvo例程详解-mqtt_tcp.md) → [OTA](./Korvo例程详解-ota.md)
