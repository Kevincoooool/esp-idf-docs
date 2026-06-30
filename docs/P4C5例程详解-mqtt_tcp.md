# P4C5 · MQTT 客户端

> **适用开发板**：P4C5 · **工程** `02.beginner.mqtt_tcp` · **target** `esp32p4`  
> **前置**：[Wi-Fi Station](./P4C5例程详解-wifi_station.md)  
> **通用概念**：[MQTT 教程](./通用-mqtt.md)

物联网设备与云端常用 **MQTT** 长连接：发布（Publish）传感器数据、订阅（Subscribe）控制指令。本例在连 Wi-Fi 后连接 Broker，自动 pub/sub，**LVGL 屏**显示 Broker、状态和收到的 Topic/Data。

---

## 1. 这个例子在干嘛

- 创建 `esp_mqtt_client`，注册 **事件回调** `mqtt_event_handler`  
- 连接成功后：向指定 Topic **发布**消息、**订阅** Topic、收到数据更新屏幕  
- 断线后客户端库 **自动重连**  

MQTT 是 **事件驱动** 的：不要在 `mqtt_event_handler` 里做长时间阻塞。

---

## 2. 编译烧录

```powershell
cd P4_C5_4.3_Firmware\02.beginner.mqtt_tcp
idf.py set-target esp32p4
idf.py menuconfig
idf.py build flash monitor
```

---

## 3. menuconfig（必配）

**Example Connection Configuration**

| 项 | 说明 |
|----|------|
| WiFi SSID / Password | 路由器 |

**Example Configuration → Broker URL**

| 默认（以当前源码为准） | 说明 |
|------------------------|------|
| `mqtt://broker-cn.emqx.io:1883` | EMQX 公共 Broker |

Topic 在 **`main/app_main.c` 源码**里写死（常见为 **`emqx/test`**）。旧 README 若写 `/topic/qos0`，**以代码为准**。

---

## 4. 运行现象

**屏幕（LVGL 仪表盘）**

- Broker 地址  
- 状态：`connected` / `subscribed` / `publish ok`  
- 最近收到的 Topic、Payload 摘要  

**串口**

```text
MQTT_EVENT_CONNECTED
sent publish successful, msg_id=...
sent subscribe successful, msg_id=...
TOPIC=emqx/test
DATA=...
MQTT_EVENT_DISCONNECTED   // 会自动重连
```

---

## 5. 用 MQTTX 交叉验证

1. 板子跑起来并 `MQTT_EVENT_CONNECTED`  
2. PC 安装 [MQTTX](https://mqttx.app/)  
3. 连接同一 Broker：`broker-cn.emqx.io:1883`  
4. 向 **`emqx/test`** 发布一条消息 → 板子屏幕/串口应收到  
5. 订阅板子 publish 的 Topic，看上行数据  

---

## 6. 关键文件

| 文件 | 作用 |
|------|------|
| `main/app_main.c` | Wi-Fi、`mqtt_app_start`、`mqtt_event_handler` |
| `main/mqtt_ui.c` | LVGL UI |
| `main/Kconfig.projbuild` | Broker URL |

---

## 7. 核心代码流程

```text
app_main()
  ├── nvs + example_connect()     // Wi-Fi，等 C5 就绪
  └── mqtt_app_start()
        ├── esp_mqtt_client_config_t { .broker.address.uri = CONFIG_* }
        ├── esp_mqtt_client_init / register_event / start

mqtt_event_handler():
  MQTT_EVENT_CONNECTED     → publish + subscribe
  MQTT_EVENT_SUBSCRIBED    → 再 publish 测试数据
  MQTT_EVENT_DATA          → 打印 topic/data，mqtt_ui 更新
  MQTT_EVENT_ERROR         → 查 TLS/socket 错误
```

### QoS 简要

- **0**：最多一次，不保证送达  
- **1**：至少一次  
- 本例用于学习，生产环境按业务选 QoS  

---

## 8. P4C5 注意

- Wi-Fi 经 **esp_hosted / esp_wifi_remote**，log 出现相关字样正常  
- 首次连接 Broker 前确保 Wi-Fi 已 `GOT IP`  
- 公共 Broker 可能限流或不稳定，量产请用自建 Mosquitto / 云 IoT  

---

## 9. 常见问题

| 现象 | 处理 |
|------|------|
| MQTT_EVENT_ERROR | Broker 地址/端口；是否需用户名密码（本例默认无） |
| 连上 Wi-Fi 但 MQTT 失败 | DNS；防火墙；Broker 是否可达 |
| 收不到 MQTTX 发的消息 | Topic 是否与源码一致 `emqx/test` |
| 屏幕无更新 | 是否连上 Broker；看串口 EVENT_DATA |

---

## 10. 动手改造

1. 改 Broker 为自己 Mosquitto：`mqtt://192.168.x.x:1883`  
2. 每 5 秒 publish 一次 `esp_get_free_heap_size()`  
3. 收到 JSON `{"cmd":"led"}` 时控制 GPIO  
4. 与 [OTA](./P4C5例程详解-ota.md) 联动：MQTT 下发 OTA URL  

---

## 11. 下一步

[HTTP Server](./P4C5例程详解-http_server.md) → [OTA](./P4C5例程详解-ota.md)

酷世DIY · Kevincoooool
