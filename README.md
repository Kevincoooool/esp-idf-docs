

          
# ESP-IDF 开发教程

## 简介

欢迎使用酷世DIY ESP-IDF 开发教程！本教程集合 ESP32 系列芯片的开发指南，从环境搭建到外设驱动、多媒体与物联网应用，并提供**两款开发板**的完整例程学习路线。

- **ESP32-S3 Korvo 2 V3**：平替乐鑫官方 Korvo 板，SPI 屏 + DVP 摄像头 + 双麦音频，60+ 例程
- **ESP32P4-C5 4.3寸触摸屏**：双芯架构（P4 主控 + C5 Wi-Fi），MIPI 大屏，108 例程

推荐开发环境：**ESP-IDF v5.5.x**（Korvo / P4C5 例程默认适配；旧教程中 5.3.x 内容仍可参考）

### 文档命名规范（`docs/`）

| 前缀 / 格式 | 含义 | 示例 |
|-------------|------|------|
| `通用-` | ESP-IDF **通用 API**，不绑定单一板子 | `通用-gpio.md`、`通用-wifi.md` |
| `Korvo例程详解-` | Korvo 仓库 **可烧录例程** 分步教程 | `Korvo例程详解-ota.md` |
| `P4C5例程详解-` | P4C5 仓库 **可烧录例程** 分步教程 | `P4C5例程详解-ota.md` |
| `*开发板介绍` / `*学习指南` | 板级说明与学习路线 | `Korvo开发板介绍.md` |

**学习顺序**：`通用-搭建编译环境` → 板子 `*学习指南` → `*例程详解-*`；`通用-*` 作 API 查阅。旧版多媒体笔记（如 `通用-avi播放`）文末已链到最新例程详解。

---

## 开发板快速入口

### ESP32-S3 Korvo 2 V3

| 文档 | 说明 |
|------|------|
| [Korvo开发板介绍](./docs/Korvo开发板介绍.md) | 硬件参数、引脚、接口 |
| [Korvo例程学习指南](./docs/Korvo例程学习指南.md) | **完整学习路线**、30 天计划、FAQ |
| [例程仓库](https://github.com/kevincoooool/KSDIY_Korvo) | `Korvo_Firmware` 源码 |

### ESP32P4-C5 4.3寸触摸屏

| 文档 | 说明 |
|------|------|
| [ESP32P4-C5开发板介绍](./docs/ESP32P4-C5开发板介绍.md) | 双芯架构、硬件参数 |
| [P4C5例程学习指南](./docs/P4C5例程学习指南.md) | **完整学习路线**、验板、多媒体 |
| [P4C5在线组件使用说明](./docs/P4C5在线组件使用说明.md) | 组件注册库引用方法 |
| [例程仓库](https://github.com/kevincoooool/ESP32P4_KSDIY) | `P4_C5_4.3_Firmware` 源码 |

### ESP32-S3 SP V4 核心板

| 文档 | 说明 |
|------|------|
| [ESP32S3 SP V4开发板介绍](./docs/ESP32S3-SP-V4开发板介绍.md) | 硬件与小智 AI 例程 |

### 通用

| 文档 | 说明 |
|------|------|
| [例程开发流程与公共组件](./docs/例程开发流程与公共组件.md) | 编译烧录、BSP、Board Manager |
| [Korvo与P4C5例程对照表](./docs/Korvo与P4C5例程对照表.md) | 两款板同名例程对照与迁移 |
| [搭建编译环境](./docs/通用-搭建编译环境.md) | Windows 下 ESP-IDF 安装 |

### 热门例程详解

按开发板分开写，**不要跨板烧录**。索引：[热门例程详解.md](./docs/热门例程详解.md)

- **Korvo**：`docs/Korvo例程详解-*.md`（如 wifi_station、spi_lcd_touch、camera_webserver、blufi…）
- **P4C5**：`docs/P4C5例程详解-*.md`（如 mipi_lcd_touch、p4c5_board_test、xiaozhi、esp-album…）
- **对照**： [Korvo与P4C5例程对照表](./docs/Korvo与P4C5例程对照表.md)（查类似功能在哪，不是逐句对比教程）

### P4C5 专题（仅 P4 仓库）

| 文档 | 说明 |
|------|------|
| [P4C5专项测试例程](./docs/P4C5专项测试例程.md) | `05.test.*` 25 项硬件/AI 测试 |
| [P4C5-GMF多媒体例程](./docs/P4C5-GMF多媒体例程.md) | `06.gmf_examples` 官方媒体管道 |

---

## 目录

### 基础入门
- [开发环境搭建](./docs/通用-搭建编译环境.md) - Windows 下 ESP-IDF 环境搭建
- [例程开发流程与公共组件](./docs/例程开发流程与公共组件.md) - 两款开发板通用开发流程
- [分区表配置](./docs/通用-partition.md) - Flash 分区表
- [Kconfig文件介绍](./docs/通用-kconfig.md) - Kconfig 配置系统
- [Menuconfig配置](./docs/通用-menuconfig.md) - menuconfig 使用
- [创建组件](./docs/通用-创建组件.md) - 自定义组件开发

### 基础外设
- [GPIO使用教程](./docs/通用-gpio.md)
- [ADC使用教程](./docs/通用-adc.md)
- [UART串口通信](./docs/通用-uart.md)
- [I2C通信教程](./docs/通用-i2c.md)
- [SPI通信教程](./docs/通用-spi.md)
- [按键使用教程](./docs/通用-button.md)

### 存储应用
- [NVS存储教程](./docs/通用-nvs.md)
- [SPIFFS文件系统](./docs/通用-spiffs.md)
- [TF卡(SD卡)使用](./docs/通用-tfcard.md)

### 网络通信
- [WiFi使用教程](./docs/通用-wifi.md)
- [配网教程](./docs/通用-配网.md)
- [HTTP客户端](./docs/通用-http.md)
- [MQTT客户端](./docs/通用-mqtt.md)

### 多媒体应用
- [音频开发教程](./docs/通用-audio.md)
- [录音播放](./docs/通用-录音播放.md)
- [摄像头驱动](./docs/通用-camera.md)
- [网页图传](./docs/通用-网页图传.md)
- [摄像头 LVGL 显示](./docs/通用-摄像头LVGL显示.md)
- [触摸屏驱动](./docs/通用-cst816.md)
- [LVGL V9](./docs/通用-lvgl-v9.md)
- [LVGL MP3 播放](./docs/通用-lvgl-mp3播放.md)
- [AVI 视频播放](./docs/通用-avi播放.md)
- [JPG下载显示](./docs/通用-jpg下载显示.md)
- [二维码识别](./docs/通用-二维码识别.md)

### 扩展功能
- [TCA9554 I/O扩展器](./docs/通用-tca9554.md)
- [USB功能实现](./docs/通用-usb.md)
- [综合硬件测试](./docs/通用-综合硬件测试.md)

### 例程专题（Korvo / P4C5）
- [Korvo与P4C5例程对照表](./docs/Korvo与P4C5例程对照表.md)
- [热门例程详解索引](./docs/热门例程详解.md) — Korvo / P4 **分篇**教程（`Korvo例程详解-*`、`P4C5例程详解-*`）
- [P4C5专项测试例程](./docs/P4C5专项测试例程.md)（05.test.*）
- [P4C5-GMF多媒体例程](./docs/P4C5-GMF多媒体例程.md)（06.gmf_examples）

---

## 学习路线建议

### Korvo S3 路线

1. [搭建编译环境](./docs/通用-搭建编译环境.md) → [Korvo例程学习指南](./docs/Korvo例程学习指南.md) 第 0～1 阶段
2. 跑通 `korvo_board_test` 或 `hello_world`，学习 [GPIO](./docs/通用-gpio.md)、[按键](./docs/通用-button.md)
3. 按指南阶段 2～4 学习外设与联网；阶段 5～8 学 LVGL、音频、USB、摄像头
4. 阶段 9～10：BSP / Board Manager → 综合项目

### P4C5 路线

1. [搭建编译环境](./docs/通用-搭建编译环境.md) → [P4C5例程学习指南](./docs/P4C5例程学习指南.md) 验板
2. 理解双芯 Wi-Fi 架构，从 `01.basic.*` 到 `02.beginner.mipi_lcd_touch`
3. 联网、USB、多媒体（AVI/MP3/摄像头）按指南阶段 3～5 推进
4. 用 [在线组件](./docs/P4C5在线组件使用说明.md) 搭建自己的工程

### 通用 API 与例程的关系

`docs/通用-*.md` 讲解 **ESP-IDF 通用 API**；`*例程详解-*.md` 提供 **板级 BSP + 可运行工程**。建议：先读通用教程理解概念，再打开对应板子的例程详解动手烧录。

---

## 开发板特点

### [ESP32S3 Korvo 2 V3](https://item.taobao.com/item.htm?id=681702043224)
- ESP32-S3 WROOM-1 N16R8（16MB Flash + 8MB PSRAM）
- 1.69 寸 ST7789 SPI 屏 + CST816 触摸
- ES8311 + ES7210 音频、双麦、3W 喇叭
- DVP 摄像头座、TF 卡、双 USB
- 60+ 分级例程，兼容官方 Korvo 2 V3

### [ESP32P4-C5 4.3寸触摸屏](https://github.com/kevincoooool/ESP32P4_KSDIY)
- ESP32-P4 + ESP32-C5 双芯（Wi-Fi 6 / BLE）
- 4.3 寸 MIPI DSI 480×800 + ST7123 触摸
- 16MB Flash + 32MB PSRAM，AXP2101 电源管理
- MIPI CSI + USB UVC 双路相机，108 分级例程
- 核心驱动已上架乐鑫组件注册库

### [ESP32S3 SP V4核心板](https://item.taobao.com/item.htm?id=850935502337)
- ESP32-S3 N16R8 + 1.69 寸触摸屏
- ES8311 音频、WS2812、TF 卡
- 支持小智 AI 等应用

---

## 注意事项

1. Korvo / P4C5 例程推荐 **ESP-IDF 5.5.x**；通用教程部分基于 5.3.x，API 大体兼容
2. 工程路径**不要含中文**；换目录或 IDF 版本时删除 `build/` 再编
3. 首次编译需联网下载 `managed_components/`
4. 硬件连接注意电平与电流，避免损坏开发板

---

## 贡献与反馈

本教程由酷世DIY团队维护。例程问题可在 GitHub 提 Issue：

- Korvo：[KSDIY_Korvo Issues](https://github.com/kevincoooool/KSDIY_Korvo/issues)
- P4C5：[ESP32P4_KSDIY Issues](https://github.com/kevincoooool/ESP32P4_KSDIY/issues)

淘宝店铺：酷世DIY

感谢您选择酷世DIY ESP-IDF 开发教程，祝您开发顺利！
