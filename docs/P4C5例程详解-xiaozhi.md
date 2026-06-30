# P4C5 · 小智 AI 语音助手

> **适用开发板**：ESP32P4-C5（**仅 P4 仓库**，Korvo 无此工程）  
> **精简版（推荐）**：`04.advanced.xiaozhi_chat-ksdiy`  
> **上游完整版**：`04.advanced.xiaozhi_ksdiy-p4c5` · [78/xiaozhi-esp32](https://github.com/78/xiaozhi-esp32)  
> **target**：`esp32p4`  
> ESP32-S3 小智见 [SP V4 开发板介绍](./ESP32S3-SP-V4开发板介绍.md)。

---

## 1. 这个例子在干嘛

**AI 语音对话助手**：本地 **WakeNet** 唤醒「你好小智」→ 麦克风采集 → **Opus** 编码推云端 → ASR/LLM/TTS → 喇叭播放 + **MIPI 大屏**聊天气泡与表情。

KSDIY 版还包含：**触摸 Wi-Fi 配网**、MCP 远程控音量/亮度/主题。

---

## 2. 两个工程怎么选

| | xiaozhi_chat-ksdiy | xiaozhi_ksdiy-p4c5 |
|--|---------------------|---------------------|
| 定位 | KSDIY **精简封装**，例程配套 | **上游完整**移植 |
| 配网 | **LVGL 触摸**扫 AP + 输密码 | BluFi 等上游方式 |
| menuconfig | 项较少 | 900+ 行，选 `Kevin P4C5 4G` |
| 教程难度 | ⭐⭐ 推荐 | ⭐⭐⭐⭐ |
| 维护 | KSDIY 例程 | 跟随上游更新 |

**初学者跑 `xiaozhi_chat-ksdiy`**；要声纹/多语言/上游 OTA 用完整版。

---

## 3. 硬件现象（KSDIY 版）

1. 启动画面「KSDIY 小智」
2. 连 Wi-Fi（120s 超时 → **左滑**进入配网页）
3. 说 **「你好小智」** → 表情变思考 → 录音上传
4. 云端回复：**喇叭 TTS** + 屏幕文字 + **21 种表情**
5. 左滑：Wi-Fi 扫描配网页

---

## 4. 编译烧录（KSDIY 版）

```powershell
cd P4_C5_4.3_Firmware\04.advanced.xiaozhi_chat-ksdiy
idf.py set-target esp32p4
idf.py menuconfig
idf.py build flash monitor
```

**注意**：
- **16MB Flash**，分区含 **model 3MB**（WakeNet 模型）
- 首次 build 拉取 **大量** managed_components，耗时长，需稳定网络
- 需烧录 **model 分区**（含 `nihaoxiaozhi` 唤醒词）

---

## 5. menuconfig（KSDIY 版）

```
KSDIY Xiaozhi Example →
    WiFi SSID / Password              可预填或配网
    Enable local wake word (WakeNet)    默认 y
    Auto Listen（跳过唤醒词）            默认 n

Xiaozhi Chat Application →
    Transport: Auto / MQTT / WebSocket

Component config → ESP Speech Recognition →
    WakeNet 模型选择
```

### 上游版额外项

```
Xiaozhi Assistant → Board Type →
    Kevin P4C5 4G (ST7102 MIPI + ML307C)   ← 必选
Default OTA URL、语言、Flash Assets…
```

---

## 6. 关键文件（KSDIY 版）

| 文件 | 作用 |
|------|------|
| `main/app_main.c` | NVS → UI → WiFi → SR 模型 → 启动聊天 |
| `main/esp_xiaozhi_chat_app.c` | Opus、云端通信、MCP |
| `main/esp_xiaozhi_chat_display.c` | TileView 聊天 + 配网 UI |
| `main/esp_xiaozhi_wifi_prov.c` | **触摸配网** |
| `components/ksdiy_board_adapter/` | P4C5 BSP → 小智 SDK |
| `partitions.csv` | factory 8MB + **model 3MB** |

---

## 7. 软件架构

```text
麦克风 ES7210 → I2S → AEC/AGC → WakeNet 唤醒
  → Opus 编码 → MQTT/WebSocket → 云端 ASR/LLM/TTS
  → Opus 解码 → ES8311 喇叭

并行：MIPI LVGL 聊天气泡 + 表情动画
      ST7123 触摸（配网 / 滑动）
      C5 esp_hosted Wi-Fi
```

### P4C5 硬件栈

| 模块 | 芯片/接口 |
|------|-----------|
| 主控 | ESP32-P4 |
| Wi-Fi | ESP32-C5 + esp_hosted |
| 屏 | ST7102 MIPI 480×800 |
| 触摸 | ST7123 |
| 音频 | ES8311 + ES7210 双麦 I2S |
| 电源 | AXP2101 |
| 背光 | GPIO6 PWM |

---

## 8. 云端配置

1. 注册 [xiaozhi.me](https://xiaozhi.me) 控制台
2. 设备启动后显示验证码，在控制台绑定
3. 配置对话模型、音色等（以官方文档为准）

传输协议 menuconfig 可选 **MQTT** 或 **WebSocket**，默认 Auto。

---

## 9. 动手改造

1. menuconfig 开 **Auto Listen**，免唤醒词直接对话
2. 改 WakeNet 唤醒词（需对应模型分区）
3. MCP 扩展：控制 GPIO/继电器
4. 配网成功后 SSID 写 NVS 下次自动连

---

## 10. 常见问题

| 现象 | 处理 |
|------|------|
| 唤醒无反应 | model 分区是否烧录；麦克风；说标准唤醒词 |
| 连不上云端 | Wi-Fi；C5 就绪；控制台是否绑定 |
| 编译组件失败 | 网络；磁盘空间；IDF 5.5.3 |
| 无声音 | ES8311；PA 使能；音量 MCP |
| 配网页空白 | 等 Wi-Fi 扫描；C5 初始化 |
| 与 Korvo SP V4 小智区别 | SP V4 用虾哥 board `kevin-sp-v4-dev`，本板为 P4C5 专用适配 |

---

## 11. 前置学习

```text
p4c5_board_test（验板）
  → wifi_station / 触摸配网理解
  → i2s_codec（音频）
  → mipi_lcd_touch（LVGL）
  → xiaozhi_chat-ksdiy
```

| 相关 | 文档 |
|------|------|
| Wi-Fi | [P4C5 wifi_station](./P4C5例程详解-wifi_station.md) |
| 配网 | Korvo 有 [BluFi](./Korvo例程详解-blufi.md)；本工程为 **触摸配网** |
| SkaiNet 测试 | [P4C5专项测试例程](./P4C5专项测试例程.md) |

酷世DIY · Kevincoooool
