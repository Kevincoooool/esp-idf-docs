# 酷世DIY ESP32P4-C5 4.3寸触摸屏开发板 · 硬件介绍

> **例程资料**：淘宝 [酷世DIY · P4C5](https://item.taobao.com/item.htm?id=667230365314) 拍下后 **联系客服**，通过 **百度网盘** 发送；解压后根目录 **`P4_C5_4.3_Firmware/`**  
> 学习指南：[P4C5例程学习指南](./P4C5例程学习指南.md)

---

## 一、产品概述

**双芯旗舰开发板**：ESP32-P4 主控 + ESP32-C5 协处理器，自带 **4.3 寸 MIPI 触控屏**，**108 个分级例程**开箱即跑。核心驱动已上架 [乐鑫 ESP 组件注册库](https://components.espressif.com)（`kevincoooool` 命名空间），支持在自己的空白工程中一键引用 BSP。

### 与 ESP32-S3 Korvo 的关键区别

| 对比项 | ESP32-S3 Korvo | ESP32P4-C5 本板 |
|--------|----------------|-----------------|
| 主控 | ESP32-S3 双核 LX7 | ESP32-P4 RISC-V，硬件 JPEG/H.264/PPA |
| 显示 | ST7789 SPI 240×280 | ST7102 **MIPI DSI** 480×800 |
| Wi-Fi | 芯片内置 | **独立 ESP32-C5**，SDIO + esp_hosted |
| Flash/PSRAM | 16MB + 8MB | **16MB + 32MB** |
| 电源管理 | AP3410 | **AXP2101**（充电、电源键、多路 DCDC） |
| 例程数量 | 60+ | **108** |

---

## 二、核心参数

### 2.1 主控与存储

| 项目 | 规格 |
|------|------|
| 主 MCU | **ESP32-P4**（双核 RISC-V，MIPI DSI/CSI、USB 2.0 HS、硬件 JPEG、PPA） |
| 协处理器 | **ESP32-C5**（Wi-Fi 6 / BLE，经 ESP-Hosted SDIO 与 P4 通信） |
| Flash | **16 MB** NOR |
| PSRAM | **32 MB** Hex，200 MHz |
| 框架 | **ESP-IDF v5.5.x**（推荐 5.5.3） |

### 2.2 显示与触摸

| 项目 | 规格 |
|------|------|
| 屏幕 | **4.3 英寸**，480×800，**MIPI DSI** 2-Lane |
| 驱动 IC | ST7102 |
| 触摸 | 电容 **ST7123**，I2C，支持 LVGL 9 输入 |
| 背光 | GPIO6，LEDC PWM 调光 |

### 2.3 音频

| 项目 | 规格 |
|------|------|
| DAC | ES8311（扬声器/耳机输出） |
| ADC | ES7210（麦克风采集） |
| 接口 | I2S 全双工 |
| 应用 | MP3、录音、FFT 频谱、AVI 音视频、SkaiNet 语音唤醒 |

### 2.4 电源与按键

| 项目 | 规格 |
|------|------|
| PMIC | AXP2101（DCDC/LDO、锂电池充电） |
| 供电 | USB Type-C |
| 按键 | 电源键（AXP2101）+ GPIO35 / GPIO0 用户键 |
| 低功耗 | Deep Sleep / 唤醒例程 |

### 2.5 存储与扩展

| 项目 | 规格 |
|------|------|
| TF 卡 | SDMMC，FAT 文件系统 |
| USB | Type-C：下载调试 + **USB Host**（UVC、MSC 等） |
| 摄像头 | **MIPI CSI**（SC2336 等）+ **USB UVC** 外接 |
| 传感器 | LSM6DS3 六轴 IMU |
| 4G | 预留 UART 例程（需另购模块） |

---

## 三、双芯架构

```
┌─────────────────────────────────────────────┐
│  ESP32-P4（主控，你的代码跑在这里）              │
│  · MIPI DSI 480×800 显示                      │
│  · MIPI CSI / USB Host                        │
│  · 硬件 JPEG / PPA / H.264                    │
│  · SDMMC / I2S / I2C / GPIO                  │
└──────────────────┬──────────────────────────┘
                   │ SDIO (ESP-Hosted)
┌──────────────────▼──────────────────────────┐
│  ESP32-C5（协处理器）                          │
│  · Wi-Fi 6 / BLE                              │
│  · 对 P4 透明：esp_wifi_remote                │
└─────────────────────────────────────────────┘
```

**开发要点**：应用代码在 P4 上运行；调用 `esp_wifi_init()` / `esp_wifi_connect()` 等与 S3 写法相同；底层由 C5 + esp_hosted 完成。首次联网需等待 C5 初始化（约 10～20 秒，log 出现 `Transport active`）。

---

## 四、关键外设引脚摘要

> 完整引脚以原理图为准；下表便于软件定位。

| 设备 | 接口 | 地址/引脚摘要 |
|------|------|----------------|
| ST7102 液晶屏 | MIPI DSI | 480×800，Reset GPIO22 |
| ST7123 触摸 | I2C0 SDA=7, SCL=8 | INT GPIO23 |
| AXP2101 电源 | I2C | 0x34 |
| ES8311 DAC | I2C + I2S | 0x18 |
| ES7210 ADC | I2C + I2S | 0x40 |
| LSM6DS3 IMU | I2C | 例程内配置 |
| SC2336 CSI | MIPI CSI + SCCB | 0x30（可选） |
| 背光 PWM | GPIO6 | LEDC |
| 用户按键 | GPIO35, GPIO0 | 低电平有效 |
| TF 卡 SDMMC | CLK=43, CMD=44, D0=39… | 见各工程 sdkconfig |
| C5 SDIO | CLK=18, CMD=19, D0~D3=14~17, RST=54 | esp_hosted 默认 |

---

## 五、接口说明

| 接口 | 功能 |
|------|------|
| USB Type-C | 供电、程序下载、串口 log |
| TF 卡槽 | 图片/视频/音频资源（FAT32） |
| 扬声器/耳机 | ES8311 音频输出 |
| 麦克风 | ES7210 采集 |
| 摄像头座 | MIPI CSI（选配 SC2336 等） |
| USB Host | UVC 摄像头、U 盘等 |
| 电源键 | AXP2101 管理开关机/唤醒 |
| 用户键 | GPIO35、GPIO0 |

---

## 六、例程与资料

| 资料 | 路径/链接 |
|------|-----------|
| 例程全集 | 百度网盘资料包内 `P4_C5_4.3_Firmware/`（108 个工程，淘宝客服发放） |
| 验板首选 | `04.advanced.p4c5_board_test` |
| 入门指南 | [P4C5例程学习指南](./P4C5例程学习指南.md) |
| 在线组件 | [P4C5在线组件使用说明](./P4C5在线组件使用说明.md) |
| 公共组件 | `components/`、`components_audio/`、`components_camera/`、`components_cellular/` |

### 例程分级

```text
01.basic.*          GPIO/UART/I2C/ADC/RTOS/NVS…
02.beginner.*       MIPI屏/Wi-Fi/USB/SD卡/音频
03.development.*    LVGL/USB/MP3/PPA显示等
04.advanced.*       摄像头/AVI/AI/相册/整板测试
04.application.*    智能面板/天气时钟等成品向
05.test.*           硬件专项测试（4G/SkaiNet/ESP-DL…）
06.gmf_examples     GMF 多媒体框架示例
```

---

## 七、典型应用场景

- 智能家居控制面板 / 工业 HMI
- 多媒体播放器（AVI/MP3/相册）
- 门禁考勤（人脸识别、二维码）
- AI 语音助手（小智等）
- 无线投屏 / USB 扩展显示器
- 物联网数据看板（MQTT + LVGL）

---

## 八、快速上手

```powershell
# 从百度网盘下载并解压到不含中文的路径后，例如：
cd D:\Firmware\P4_C5_4.3_Firmware\04.advanced.p4c5_board_test

idf                          # 导入 ESP-IDF 5.5.x 环境
idf.py set-target esp32p4    # 首次必须
idf.py build flash monitor
```

屏幕出现 **NODE P4** 赛博风格界面即验板成功。

---

酷世DIY · Kevincoooool
