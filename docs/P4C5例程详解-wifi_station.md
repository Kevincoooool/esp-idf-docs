# P4C5 · Wi-Fi Station 联网

> **适用开发板**：ESP32P4-C5 4.3 寸触摸屏  
> **工程**：`P4_C5_4.3_Firmware/02.beginner.wifi_station`  
> **target**：`esp32p4`  
> **API 概念**：[WiFi使用教程](./通用-wifi.md)

Station 模式连接路由器、DHCP 获 IP；屏幕显示 SSID、连接状态、IP 与重试次数。后续 HTTP/MQTT/OTA 都建立在本例之上。

**与 Korvo 最大区别**：Wi-Fi 不在 P4 主控上，而在板载 **ESP32-C5 协处理器**，经 **SDIO + esp_hosted** 暴露给 P4；应用层仍调用 `esp_wifi_*`，但上电需 **等 C5 就绪**。

---

## 1. 这个例子在干嘛

```text
app_main
  → display / LVGL 状态卡
  → nvs_flash_init
  → wifi_init_sta()
       STA_START → esp_wifi_connect()
       DISCONNECTED → 重连 + 屏幕显示 retry 次数
       GOT_IP → 屏幕显示 IP
```

Wi-Fi 是 **事件驱动**，不要在 `app_main` 里 `while` 阻塞等 IP。

---

## 2. 编译烧录

```powershell
cd F:\Github\ESP32P4_KSDIY\P4_C5_4.3_Firmware\02.beginner.wifi_station
idf.py set-target esp32p4
idf.py menuconfig
idf.py build flash monitor
```

---

## 3. menuconfig

**Example Connection Configuration**（部分工程菜单名略有不同，以 Kconfig 为准）：

| 项 | 说明 |
|----|------|
| WiFi SSID | 路由器名称 |
| WiFi Password | 密码 |
| Maximum retry | 最大重连次数 |

---

## 4. P4C5 联网要点

| 要点 | 说明 |
|------|------|
| C5 启动时间 | 上电后约 **10～20 s** 再判断失败；log 出现 **`Transport active`** 表示链路 OK |
| 底层栈 | `esp_wifi_remote` / `esp_hosted`，log 里出现 hosted 字样正常 |
| 频段 | 仅 **2.4 GHz** |
| 隔离测试 | 只验 C5 可先跑 **`05.test.wifi_scan_c5`** |

---

## 5. 运行现象

**屏幕（约三行）**

- SSID / 连接中 / 已连接  
- IP 地址（获 IP 后）  
- 重试次数（失败时）  

**串口**

```text
wifi:connected with ...
wifi:got ip:192.168.x.x
```

---

## 6. 关键文件

| 文件 | 作用 |
|------|------|
| `main/station_example_main.c` | Wi-Fi 事件、重连逻辑 |
| `ksdiy_p4c5_bsp` / display 组件 | 屏幕 UI |

---

## 7. 常见问题

| 现象 | 处理 |
|------|------|
| 上电立刻失败 | **等 C5**；勿 5 s 内就结论 |
| 一直 retry | SSID/密码；2.4G；隐藏 SSID |
| P4 正常但无 Wi-Fi | ESP-Hosted 分区；C5 协固件版本 |
| 连上无 IP | 路由器 DHCP；MAC 过滤 |
| 改 SSID 无效 | menuconfig 保存后 **重新 build flash** |

---

## 8. 动手改造

1. 获 IP 后屏幕显示 RSSI  
2. 失败超过 N 次显示「请检查路由器」  
3. 对比 `05.test.wifi_scan_c5` 扫描列表  

---

## 9. 下一步（P4 仓库）

[HTTP Server](./P4C5例程详解-http_server.md) → [MQTT](./P4C5例程详解-mqtt_tcp.md) → [OTA](./P4C5例程详解-ota.md)

酷世DIY · Kevincoooool
