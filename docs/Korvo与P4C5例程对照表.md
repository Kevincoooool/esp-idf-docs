# Korvo 与 P4C5 例程对照表

> **用途**：查「另一块板有没有类似功能的工程、仓库路径叫什么」——**不是**把两篇教程合并成对比阅读。  
> 实际操作请只看自己板子下的 [Korvo例程详解-*](./热门例程详解.md#korvo-s3-例程详解) 或 [P4C5例程详解-*](./热门例程详解.md#p4c5-例程详解)。

---

## 一、硬件与开发差异速查

| 对比项 | Korvo S3 | P4C5 |
|--------|----------|------|
| set-target | `esp32s3` | `esp32p4` |
| 推荐 IDF | 5.5.x | 5.5.3 |
| 显示 | ST7789 SPI 240×280 | ST7102 MIPI 480×800 |
| 触摸 | CST816S | ST7123 |
| Wi-Fi | S3 内置，即连即用 | C5 协处理器，等 10～20s |
| 摄像头 | DVP 24P | MIPI CSI + USB UVC |
| 视频解码 | 软件 JPEG 为主 | **硬件 JPEG + PPA** |
| 电源 | AP3410 | **AXP2101**（电池/充电） |
| 按键 | ADC 六键 GPIO5 | GPIO35 + GPIO0 |
| IO 扩展 | TCA9554 背光 | GPIO6 PWM 背光 |
| IMU | 无 | LSM6DS3 |
| 公共 BSP | `ksdiy_korvo_bsp` | `ksdiy_p4c5_bsp` |
| 统一 UI 组件 | `ksdiy_example_display` | 同模式 + `ksdiy_lvgl_port_init()` |
| 整板测试 | `korvo_board_test` | `p4c5_board_test` |
| 例程总数 | 74（01～04） | 108 + 24 GMF + 24 test |

---

## 二、01.basic 基础层对照

| 功能 | Korvo | P4C5 | 备注 |
|------|-------|------|------|
| hello_world | ✅ | ✅ | 相同学习目的 |
| blink / gpio | ✅ | ✅ | P4 带 WS2812 |
| uart_echo | ✅ | ✅ | |
| i2c | ✅ | ✅ | 板上设备地址不同 |
| adc_oneshot / continuous | ✅ | ✅ | |
| ledc_pwm / gptimer / rmt | ✅ | ✅ | |
| nvs / spiffs / fatfs | ✅ | ✅ | P4 多 `littlefs` |
| freertos / deep_sleep | ✅ | ✅ | P4 低功耗与 AXP 相关 |

**P4C5 独有**：`01.basic.littlefs`

---

## 三、02.beginner 入门层对照

| 功能 | Korvo | P4C5 | 差异说明 |
|------|-------|------|----------|
| wifi_scan | ✅ | — | P4 用 `05.test.wifi_scan_c5` |
| wifi_station | ✅ | ✅ | Korvo：[Korvo wifi_station](./Korvo例程详解-wifi_station.md) · P4：[P4 wifi_station](./P4C5例程详解-wifi_station.md) |
| wifi_softap | ✅ | ✅ | |
| http_request / http_server | ✅ | ✅ | |
| mqtt_tcp / ota / blufi | ✅ | ✅ | |
| ble_spp_server | ✅ | ✅ | |
| **屏幕+触摸入门** | `spi_lcd_touch` | `mipi_lcd_touch` | [Korvo SPI 屏](./Korvo例程详解-spi_lcd_touch.md) · [P4 MIPI 屏](./P4C5例程详解-mipi_lcd_touch.md) |
| sdmmc_sd | ✅ | ✅ | |
| i2s_codec | ✅ | ✅ | 同为 ES8311+ES7210 |
| usb_cdc / msc / hid | ✅ | ✅ | Korvo 用右 USB；P4 用 Type-C Host |
| sdmmc | — | ✅（02 层） | |

---

## 四、03.development 开发层对照

| 功能 | Korvo | P4C5 | 备注 |
|------|-------|------|------|
| lvgl_esp_port / adapter / v9 | ✅ | ✅ | P4 多 `lvgl_v8`、`lvgl_esp_adapter_rotate` |
| lvgl_png / jpg / freetype / rlottie | ✅ | ✅ | |
| lvgl_adc_button | ✅ | ✅ | P4 多 `lvgl_gpio_button_component` |
| lvgl_eez / squareline | ✅ | ✅ | |
| mp3_player | ✅ | ✅ | [Korvo MP3](./Korvo例程详解-mp3_player.md) · [P4 MP3](./P4C5例程详解-mp3_player.md) |
| audio_fft / record_play / record_sdcard | ✅ | ✅ | |
| photo_album | ✅ | ✅ | |
| usb_cdc_msc / rndis / wireless_disk / 4g | ✅ | ✅ | |
| usb_audio_player | ✅ | ✅ | |
| low_power_wakeup | ✅ | ✅ | |
| **PPA 显示** | — | `p4c5_ppa_display` | P4 独有 |
| **LSM6DS3** | — | `lvgl_lsm6ds3` | P4 独有 |
| codex_quota_lvgl_adapter | — | ✅ | P4 独有 |
| usb_msc_ota | — | ✅ | P4 独有 |

---

## 五、04.advanced 进阶层对照

| 功能 | Korvo | P4C5 | 备注 |
|------|-------|------|------|
| **整板测试** | `korvo_board_test` | `p4c5_board_test` | [Korvo](./Korvo例程详解-korvo_board_test.md) · [P4](./P4C5例程详解-p4c5_board_test.md) |
| avi_player | ✅ | ✅ | [Korvo SPI 软解](./Korvo例程详解-avi_player.md) · [P4 硬解](./P4C5例程详解-avi_player.md) |
| avi_multi_player / avi_recorder | ✅ | ✅ | P4 多 `avi_player_lvgl` |
| camera_lcd / lvgl / webserver | ✅ | ✅ | Korvo DVP；P4 CSI+UVC |
| qrcode / face / color_tracking | ✅ | ✅ | |
| spectrum_box_lite | ✅ | ✅ | |
| usb_camera / extended_display | ✅ | ✅ | |
| korvo_bsp_demo / board_manager | ✅ | — | P4 用 `p4c5_board_manager` |
| **mjpeg_album** | — | ✅ | P4 独有 |
| **esp-album-ksdiy** | — | ✅ | P4 独有 |
| **xiaozhi / airplay** | — | ✅ | P4 独有 |
| **esp_claw / wifi_av_link** | — | ✅ | P4 独有 |
| p4c5_board_test_sim | — | ✅ | PC 侧仿真 UI |

**P4C5 独有 04 层**（Korvo 无对应）：

| 工程 | 功能 |
|------|------|
| `04.application.ksdiy_smart_panel` | 智能面板成品向 |
| `04.application.weather_clock` | 天气时钟 |
| `04.application.uart_logger` | 串口日志器 |
| `04.advanced.cloud_limited_programmer` | 云烧录 |
| `04.advanced.p4c5_blufi_mqtt_http_avi` | 联网+多媒体综合 |

---

## 六、P4C5 独有目录（Korvo 无）

### 05.test.* 专项测试（24 个）

详见 [P4C5专项测试例程](./P4C5专项测试例程.md)（25 个）

| 分类 | 代表工程 |
|------|----------|
| 硬件验证 | axp2101、codec_i2s、sdmmc、button_ws2812 |
| C5 联网 | wifi_scan_c5、wifi_station_c5 |
| USB/视觉 | usb_uvc_display、usb_qr_recognition |
| 4G | 4g_module、4g_http、4g_mqtt |
| SkaiNet 语音 | skainet_afe_wakeup、skainet_cn_cmd、skainet_doa_display… |
| ESP-DL | espdl_cat_detect、espdl_mobilenet_cls、espdl_speaker… |
| 其他 | airplay_p4c5、rs485_uart、mcp4725_dac |

### 06.gmf_examples GMF 多媒体（24 个）

详见 [P4C5-GMF多媒体例程](./P4C5-GMF多媒体例程.md)

---

## 七、按学习目标选板与例程

| 我想学… | 推荐板 | 入门例程 |
|---------|--------|----------|
| ESP-IDF 基础外设 | 均可 | `01.basic.*` |
| SPI 小屏 LVGL | **Korvo** | `02.beginner.spi_lcd_touch` |
| MIPI 大屏 LVGL | **P4C5** | `02.beginner.mipi_lcd_touch` |
| Wi-Fi 物联网 | Korvo 更简单 | `02.beginner.wifi_station` |
| 理解双芯 Wi-Fi | **P4C5** | `05.test.wifi_scan_c5` → `wifi_station` |
| MP3 播放器 | 均可 | `03.development.mp3_player` |
| AVI 视频（性能） | **P4C5** | `04.advanced.avi_player` |
| DVP 摄像头 | **Korvo** | `04.advanced.camera_lvgl_display` |
| CSI + UVC 双相机 | **P4C5** | `p4c5_board_test` 相机页 |
| 官方 GMF 管道 | **P4C5** | `06.gmf_examples/pipeline_play_embed_music` |
| Board Manager | Korvo | `04.advanced.korvo_board_manager_demo` |
| 平替乐鑫 Korvo 官方例程 | **Korvo** | 引脚兼容 ESP32-S3-Korvo-2 |

---

## 八、迁移注意事项

### 从 Korvo 迁到 P4C5

1. `idf.py set-target esp32p4`，不能沿用 esp32s3 的 build 目录
2. 所有 `ksdiy_korvo_*` / `ksdiy_example_display` 改为 `ksdiy_p4c5_bsp` + `ksdiy_lvgl_port_init()`
3. Wi-Fi 例程需等待 C5 就绪；可先看 `05.test.wifi_scan_c5`
4. 显示从 SPI flush 改为 MIPI；AVI 可改用硬件 JPEG 路径
5. 不再有 TCA9554；背光用 GPIO6 PWM
6. ADC 六键逻辑不适用，改用 GPIO35/GPIO0

### 从 P4C5 迁到 Korvo

1. `set-target esp32s3`
2. MIPI/LVGL 9 大屏代码需改为 SPI ST7789（240×280）
3. Wi-Fi 去掉 esp_hosted 等待逻辑
4. AVI 无 PPA，分辨率建议 280×240 MJPEG
5. 摄像头从 CSI 改为 DVP 接口与引脚

---

## 九、相关文档

| 文档 | 说明 |
|------|------|
| [Korvo例程学习指南](./Korvo例程学习指南.md) | Korvo 完整路线 |
| [P4C5例程学习指南](./P4C5例程学习指南.md) | P4C5 完整路线 |
| [例程开发流程与公共组件](./例程开发流程与公共组件.md) | 通用编译与 BSP |
| [热门例程详解索引](./热门例程详解.md) | 单例程深度教程 |

酷世DIY · Kevincoooool
