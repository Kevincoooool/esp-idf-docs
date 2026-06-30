# Korvo · MQTT 客户端

> **适用开发板**：Korvo S3 · **工程** `02.beginner.mqtt_tcp` · **target** `esp32s3`  
> **前置**：[Wi-Fi Station](./Korvo例程详解-wifi_station.md)  
> **概念**：[MQTT 教程](./通用-mqtt.md)

连 Wi-Fi 后创建 `esp_mqtt_client`，在事件回调里完成发布、订阅与收消息；屏幕显示 Broker 与状态。S3 **内置 Wi-Fi**，上电即可连网，无 P4 的 C5 等待。

---

## 1. 这个例子在干嘛

- **MQTT_EVENT_CONNECTED** → publish + subscribe 多个 Topic  
- **MQTT_EVENT_SUBSCRIBED** → 再 publish 测试数据  
- **MQTT_EVENT_DATA** → 打印 topic/payload，更新屏幕  
- **MQTT_EVENT_DISCONNECTED** → 库自动重连  

逻辑集中在 `mqtt_event_handler`，勿在回调里长时间阻塞。

---

## 2. 编译烧录

```powershell
cd Korvo_Firmware\02.beginner.mqtt_tcp
idf.py set-target esp32s3
idf.py menuconfig
idf.py build flash monitor
```

---

## 3. menuconfig

**Example Connection Configuration** → WiFi SSID / Password  

**Example Configuration → Broker URL**

| 默认（以源码为准） | `mqtt://mqtt.eclipseprojects.io` |
|--------------------|----------------------------------|

Topic 名称在 **`main/app_main.c` 源码**里定义（常见 `/topic/qos0`、`/topic/qos1`），**以代码为准**，不要只看旧注释。

---

## 4. 运行现象

**屏幕**：Broker 地址、connected/subscribed/publish ok、最近 Topic/Data  

**串口**：

```text
MQTT_EVENT_CONNECTED
sent publish successful, msg_id=...
TOPIC=/topic/qos0
DATA=...
```

---

## 5. MQTTX 验证

1. 板子 `CONNECTED`  
2. PC 打开 MQTTX，连同一 Broker  
3. 向源码里的 Topic 发消息 → 板子应收并显示  
4. 订阅板子 publish 的 Topic 看上行  

---

## 6. 关键文件

| 文件 | 作用 |
|------|------|
| `main/app_main.c` | Wi-Fi、mqtt 客户端、事件处理 |
| display 组件 | 屏幕 UI |

---

## 7. 与 P4C5 差异（仅环境）

| | Korvo | P4C5 |
|--|-------|------|
| Wi-Fi | S3 内置 | C5 协处理器 |
| 默认 Broker | eclipseprojects | emqx（以各工程为准） |
| UI | 小屏 SPI | MIPI 大屏 |

API 用法相同，**bin 不可混烧**。

---

## 8. 常见问题

| 现象 | 处理 |
|------|------|
| MQTT_EVENT_ERROR | Broker 地址；是否需要 TLS/账号 |
| 收不到消息 | Topic 与源码一致 |
| 公共 Broker 不稳定 | 换自建 Mosquitto |

---

## 9. 下一步

[OTA](./Korvo例程详解-ota.md) · [HTTP Server](./Korvo例程详解-http_server.md)

P4 版 → [P4C5 MQTT](./P4C5例程详解-mqtt_tcp.md)。

酷世DIY · Kevincoooool
