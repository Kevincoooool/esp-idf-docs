# 酷世DIY ESP32S3 Korvo 2 V3 · 例程学习指南

> 例程目录 **`Korvo_Firmware/`** — 淘宝 [拍下 Korvo 板](https://item.taobao.com/item.htm?id=681702043224) 后 **联系客服**，**百度网盘** 发全套资料  
> **视频系列 K**：[文档总目录与视频教程大纲](./文档总目录与视频教程大纲.md#七系列-k--korvo-实战按学习指南顺序)  
> 硬件介绍见 [Korvo开发板介绍](./Korvo开发板介绍.md)

---

## 一、拿到板子后先做什么

### 1.0 获取例程（必读）

1. 淘宝 **酷世DIY** 购买 Korvo 开发板  
2. **下单后联系客服**，索取例程资料  
3. 客服通过 **百度网盘** 发送压缩包（含 `Korvo_Firmware/` 及说明）  
4. 解压到**无中文路径**（如 `D:\Firmware\`）

### 1.1 开箱检查

| 物品 | 说明 |
|------|------|
| Korvo 开发板 | 确认屏幕、排针、接口无损伤 |
| USB Type-C 数据线 × 2 | 左口下载/日志，右口 ESP32-S3 原生 USB |
| TF 卡（可选） | 多媒体例程需要，格式化为 FAT32 |
| 例程资料 | 淘宝下单后 **联系客服**，**百度网盘** 发链接 |

### 1.2 第一次验板（推荐）

```powershell
# 百度网盘下载解压后（路径不要含中文）
cd D:\Firmware\Korvo_Firmware

# 导入 IDF 环境后
cd 04.advanced.korvo_board_test
idf.py set-target esp32s3    # 首次必须执行
idf.py build flash monitor
```

**预期现象**：屏幕进入整板测试界面，可逐项检测 LCD、触摸、ADC 按键、音频、摄像头、Wi-Fi 等。

若只想最快确认环境，可先跑 `01.basic.hello_world`——屏幕会显示统一状态页，串口输出芯片信息。

### 1.3 两个 USB 口别搞混

| 接口 | 用途 |
|------|------|
| **左 USB**（CH9102F） | 程序下载、串口日志，日常开发用这个 |
| **右 USB**（ESP32-S3 内置） | USB Device/Host：CDC、MSC、HID、UVC、4G 模块等 |

---

## 二、例程仓库结构

```text
Korvo_Firmware/
├── 01.basic.*          基础外设（GPIO/UART/I2C/ADC/RTOS/NVS…）
├── 02.beginner.*       联网、USB、屏幕触摸、SD 卡、音频
├── 03.development.*    LVGL、音频、USB 复合、相册、MP3 等
├── 04.advanced.*       摄像头、视觉、AVI、整板测试、Board Manager
├── components/         公共组件（显示 UI、BSP、板卡描述）
└── docs/tutorials/     仓库内配套教程（与本指南互补）
```

**命名规则**：`层级.分类.工程名`，例如 `02.beginner.wifi_station` 表示「入门级 · Wi-Fi Station 连接」。

当前仓库约 **60+** 个独立 ESP-IDF 工程，每个目录都是完整工程，切换例程需要 `cd` 到对应目录再 `idf.py build`。

---

## 三、公共组件（必读）

Korvo 外设多，仓库用三类公共能力避免每个例程重复写初始化代码：

| 组件 | 路径 | 用途 |
|------|------|------|
| `ksdiy_example_display` | `components/` | 普通例程的统一屏幕 UI（标题 + 三张状态卡） |
| `ksdiy_korvo_bsp` | `components/` | 板级 BSP：LCD、触摸、音频、摄像头、ADC 按键 |
| `ksdiy_korvo_s3` | `components/` | ESP Board Manager 板卡 YAML 描述 |

### 3.1 普通例程的屏幕 UI

大多数 `01.basic.*` 和 `02.beginner.*` 例程启动时会调用：

```c
ksdiy_example_display_bootstrap("01.basic.hello_world", "chip info and restart");
ksdiy_example_display_set_lines("line1", "line2", "line3");
```

页面结构：顶部标题栏 + `STATE` / `DATA` / `DETAIL` 三张状态卡。业务逻辑在 `main/`，UI 在公共组件里维护。

**不适合**套此 UI 的工程：完整 LVGL Demo、摄像头实时预览、视频播放器、整板测试——它们自带完整界面。

### 3.2 做自己的产品时用 BSP

```c
ksdiy_korvo_audio_config_t audio_cfg = KSDIY_KORVO_AUDIO_DEFAULT_CONFIG();
ksdiy_korvo_audio_init(&audio_cfg);
ksdiy_korvo_buttons_init(button_cb, NULL);
```

参考工程：`04.advanced.korvo_bsp_demo`

### 3.3 接乐鑫官方例程用 Board Manager

```powershell
cd 04.advanced.korvo_board_manager_demo
idf.py bmgr -b ..\components\ksdiy_korvo_s3
idf.py build flash monitor
```

应用里通过设备名获取 handle：`display_lcd`、`lcd_touch`、`camera`、`audio_dac`、`audio_adc`、`adc_button_group`。

更多细节见 [例程开发流程与公共组件](./例程开发流程与公共组件.md)。

---

## 四、全系列学习路线

建议节奏：**先跑起来 → 改出变化 → 再理解原理**。不要追求一次看懂所有代码。

### 总体路线图

```text
第 0 阶段：认识开发板和环境
  ↓
第 1 阶段：ESP-IDF 入门 + 屏幕状态页
  ↓
第 2 阶段：GPIO / UART / I2C / ADC / PWM / Timer
  ↓
第 3 阶段：存储、FreeRTOS、低功耗
  ↓
第 4 阶段：Wi-Fi / HTTP / MQTT / OTA
  ↓
第 5 阶段：LCD / 触摸 / LVGL / 图片 / 字体 / 动画
  ↓
第 6 阶段：音频输入输出 / 录音 / FFT / MP3
  ↓
第 7 阶段：USB CDC / MSC / HID / 复合设备
  ↓
第 8 阶段：摄像头 / 网页预览 / 视觉算法
  ↓
第 9 阶段：BSP 与 ESP Board Manager
  ↓
第 10 阶段：综合项目与二次开发
```

---

### 第 0 阶段：认识硬件

| 模块 | 型号/接口 | 典型例程 |
|------|-----------|----------|
| 主控 | ESP32-S3 N16R8 | 全部 |
| 屏幕 | ST7789 SPI 240×280 | LVGL、状态页、摄像头显示 |
| 触摸 | CST816S I2C | LVGL、触摸例程 |
| 音频输出 | ES8311 | I2S、MP3、语音播放 |
| 音频输入 | ES7210 多麦 | 录音、FFT、语音识别 |
| 摄像头 | DVP 24P | 网页图传、二维码、人脸 |
| 按键 | ADC 六键 GPIO5 | 菜单、整板测试 |
| USB | Device + Host | CDC、MSC、HID、UVC |
| IO 扩展 | TCA9554 | 背光控制 |

**过关标准**：能说出各外设接口类型；知道 `01`~`04` 目录含义；知道公共组件在 `components/`。

---

### 第 1 阶段：ESP-IDF 入门

| 顺序 | 工程 | 学习重点 |
|------|------|----------|
| 1 | `01.basic.hello_world` | 工程结构、日志、芯片信息、屏幕状态页 |
| 2 | `01.basic.blink` | GPIO 输出、循环任务 |
| 3 | `01.basic.uart_echo` | 串口收发、调试 |

```powershell
cd Korvo_Firmware\01.basic.hello_world
idf.py build flash monitor
```

**必看文件**：
- `main/hello_world_main.c` 中的 `app_main()`
- `components/ksdiy_example_display/ksdiy_example_display.c`

**动手作业**：
1. 把屏幕标题改成自己的名字
2. 把 blink 闪烁周期改成 200ms
3. uart_echo 收到 `help` 时额外打印说明

**过关标准**：独立编译烧录；能从串口判断程序运行位置；能改屏幕上一行文字。

---

### 第 2 阶段：基础外设

| 顺序 | 工程 | 学习重点 | 建议时间 |
|------|------|----------|----------|
| 1 | `01.basic.gpio` | 输入、输出、中断 | 30 分钟 |
| 2 | `01.basic.i2c` | I2C 扫描、触摸芯片 | 1 小时 |
| 3 | `01.basic.adc_oneshot` | ADC 单次采样 | 30 分钟 |
| 4 | `01.basic.adc_continuous` | ADC 连续采样、DMA | 1 小时 |
| 5 | `01.basic.ledc_pwm` | PWM 频率、占空比 | 30 分钟 |
| 6 | `01.basic.gptimer` | 硬件定时器回调 | 45 分钟 |
| 7 | `01.basic.rmt_led_strip` | RMT 精确时序 | 45 分钟 |

**板上关联**：I2C 连触摸/Codec/TCA9554；ADC 读六键；LEDC 可做背光 PWM。

**动手作业**：gpio 把中断次数显示到 STATE 卡；i2c 把扫描地址显示到 DATA 卡；adc 换算成百分比。

---

### 第 3 阶段：存储与系统

| 顺序 | 工程 | 学习重点 |
|------|------|----------|
| 1 | `01.basic.nvs` | 键值配置存储 |
| 2 | `01.basic.spiffs` | SPI Flash 文件系统 |
| 3 | `01.basic.fatfs` | FAT 分区 |
| 4 | `02.beginner.sdmmc_sd` | TF 卡读写 |
| 5 | `01.basic.freertos` | 任务、队列、锁 |
| 6 | `01.basic.deep_sleep` | 深睡眠与唤醒 |

**要点**：NVS 存小配置；SPIFFS 放资源；SD 卡放大文件（音频/照片/视频）；FreeRTOS 让多任务并行。

---

### 第 4 阶段：联网与 OTA

| 顺序 | 工程 | 学习重点 |
|------|------|----------|
| 1 | `02.beginner.wifi_scan` | 扫描 AP |
| 2 | `02.beginner.wifi_station` | 连接路由器 |
| 3 | `02.beginner.wifi_softap` | 开热点 |
| 4 | `02.beginner.http_request` | 客户端请求 |
| 5 | `02.beginner.http_server` | 板载 Web 服务 |
| 6 | `02.beginner.mqtt_tcp` | MQTT 发布订阅 |
| 7 | `02.beginner.ota` | 空中升级 |
| 8 | `02.beginner.blufi` | BLE 配网 |

Wi-Fi 连接是**事件驱动**的，不是阻塞等待。HTTP request 是板子访问别人；HTTP server 是别人访问板子。

相关通用教程：[WiFi使用教程](./通用-wifi.md)、[HTTP客户端](./通用-http.md)、[MQTT客户端](./通用-mqtt.md)、[配网教程](./通用-配网.md)

---

### 第 5 阶段：LVGL 与屏幕

| 顺序 | 工程 | 学习重点 |
|------|------|----------|
| 1 | `02.beginner.spi_lcd_touch` | LCD + 触摸 + LVGL 最小闭环 |
| 2 | `03.development.lvgl_esp_port` | LVGL 移植 |
| 3 | `03.development.lvgl_esp_adapter` | 乐鑫适配组件 |
| 4 | `03.development.lvgl_v9` | LVGL 9 基础 |
| 5 | `03.development.lvgl_png` | PNG 图片 |
| 6 | `03.development.lvgl_spiffs_jpg` | SPIFFS JPG |
| 7 | `03.development.lvgl_sdcard_jpg` | SD 卡 JPG |
| 8 | `03.development.lvgl_freetype` | TTF 字体 |
| 9 | `03.development.lvgl_qrcode_font` | 中文 + 二维码 |
| 10 | `03.development.lvgl_rlottie` | Lottie 动画 |
| 11 | `03.development.lvgl_eez_demo` | EEZ Studio |
| 12 | `03.development.lvgl_squareline_demo` | SquareLine Studio |

相关教程：[LVGL V9](./通用-lvgl-v9.md)、[CST816 触摸](./通用-cst816.md)

---

### 第 6 阶段：音频

| 顺序 | 工程 | 学习重点 |
|------|------|----------|
| 1 | `02.beginner.i2s_codec` | ES8311/ES7210 + I2S |
| 2 | `03.development.audio_record_play` | 录音回放 |
| 3 | `03.development.audio_record_sdcard` | 录音存 SD |
| 4 | `03.development.audio_fft` | 音频频谱 |
| 5 | `03.development.mp3_player` | MP3 播放器 |
| 6 | `03.development.usb_audio_player` | USB 音频 |
| 7 | `04.advanced.spectrum_box_lite` | 频谱可视化综合 |

**I2S 信号**：MCLK、BCLK、WS、DIN、DOUT——详见 [音频开发教程](./通用-audio.md)、[录音播放](./通用-录音播放.md)

---

### 第 7 阶段：USB

| 顺序 | 工程 | 学习重点 |
|------|------|----------|
| 1 | `02.beginner.usb_cdc` | 虚拟串口 |
| 2 | `02.beginner.usb_msc_ram` | RAM U 盘 |
| 3 | `02.beginner.usb_msc_flash` | Flash U 盘 |
| 4 | `02.beginner.usb_hid_mouse` | HID 鼠标 |
| 5 | `02.beginner.usb_hid_keyboard` | HID 键盘 |
| 6 | `03.development.usb_cdc_msc` | CDC + MSC 复合 |
| 7 | `03.development.usb_rndis` | USB 网卡 |
| 8 | `03.development.usb_4g_module` | 4G 模块 |
| 9 | `04.advanced.usb_camera_display` | USB 摄像头 |
| 10 | `04.advanced.usb_extended_display` | USB 扩展屏 |

使用**右 USB 口**。详见 [USB功能实现](./通用-usb.md)

---

### 第 8 阶段：摄像头与视觉

| 顺序 | 工程 | 学习重点 |
|------|------|----------|
| 1 | `04.advanced.camera_lcd_display` | 摄像头直显 LCD |
| 2 | `04.advanced.camera_lvgl_display` | 摄像头 + LVGL |
| 3 | `04.advanced.camera_webserver` | 网页预览 |
| 4 | `04.advanced.color_tracking_lvgl` | 颜色追踪 |
| 5 | `04.advanced.qrcode_detection` | 二维码 |
| 6 | `04.advanced.face_recognition` | 人脸识别 |

相关教程：[摄像头驱动](./通用-camera.md)、[网页图传](./通用-网页图传.md)、[LCD显示摄像头结合lvgl](./通用-摄像头LVGL显示.md)、[二维码识别](./通用-二维码识别.md)

---

### 第 9 阶段：BSP 与 Board Manager

| 顺序 | 内容 | 工程/路径 |
|------|------|-----------|
| 1 | 公共 UI 原理 | `components/ksdiy_example_display` |
| 2 | BSP API | `components/ksdiy_korvo_bsp` |
| 3 | BSP 演示 | `04.advanced.korvo_bsp_demo` |
| 4 | 板卡 YAML | `components/ksdiy_korvo_s3` |
| 5 | Board Manager 演示 | `04.advanced.korvo_board_manager_demo` |

---

### 第 10 阶段：综合项目

| 工程 | 综合能力 |
|------|----------|
| `03.development.photo_album` | 文件系统 + 图片 + LVGL |
| `03.development.mp3_player` | 文件系统 + 音频 + UI |
| `04.advanced.camera_webserver` | 摄像头 + Wi-Fi + HTTP |
| `04.advanced.korvo_board_test` | 整板硬件测试 |
| `04.advanced.spectrum_box_lite` | 音频 + FFT + UI |
| `04.advanced.avi_player` | AVI 视频播放 |
| `04.advanced.avi_recorder` | AVI 视频录制 |

**读综合项目的方法**：先画模块图（`app_main` → 硬件初始化 → UI → 任务 → 数据流 → 更新界面），不要从第一行硬啃。

---

## 五、日常开发工作流

### 5.1 切换例程

每个子目录是独立工程，必须 `cd` 进去再编译：

```powershell
cd Korvo_Firmware\03.development.lvgl_v9
idf.py build flash monitor
```

### 5.2 常用命令

```powershell
idf.py build              # 编译
idf.py flash              # 烧录
idf.py monitor            # 串口监视（Ctrl+] 退出）
idf.py -p COM5 flash monitor   # 指定端口
idf.py -b 2000000 flash        # 提高波特率加速下载
idf.py build app-flash         # 只烧 app 分区
idf.py menuconfig              # 配置 SSID、分区表等
idf.py fullclean               # 清 build 后重编
```

### 5.3 读 log 技巧

- `I (xxx)`：正常信息
- `W (xxx)`：警告，一般可继续
- `E (xxx)`：错误，需处理
- `Guru Meditation` / `panic`：严重错误，复制完整 backtrace 再排查

### 5.4 注意事项

- 工程路径**不要有中文**
- 换路径或换 IDF 版本时，删除 `build/` 再编
- 首次 build 需联网下载 `managed_components/`
- 杀毒软件会拖慢编译，建议临时关闭

---

## 六、30 天学习计划（参考）

| 天数 | 内容 | 产出 |
|------|------|------|
| Day 1 | 环境 + `korvo_board_test` 或 `hello_world` | 验板通过 |
| Day 2～3 | 01.basic 选读（gpio/uart/i2c/adc） | 理解基础外设 |
| Day 4～5 | spi_lcd_touch + lvgl_v9 | 自建 Label/Button 界面 |
| Day 6～7 | sdmmc_sd + lvgl_sdcard_jpg | TF 卡相册 |
| Day 8～9 | i2s_codec + audio_fft + mp3_player | 音频 + 频谱 |
| Day 10～12 | wifi_station + http + mqtt | 联网显示数据 |
| Day 13～14 | usb_cdc + usb_hid_keyboard | USB 外设 |
| Day 15～17 | lvgl 组件选 3 个（png/freetype/rlottie） | 丰富 UI |
| Day 18～20 | camera_webserver 或 avi_player | 视觉/视频 |
| Day 21～30 | 自选 advanced 项目 | 完整小作品 |

---

## 七、常见问题 FAQ

### Q1：屏幕亮了但触摸没反应？

确认例程使用 LVGL 触摸 indev 初始化；跑 `korvo_board_test` 硬件检测页；查看 log 中 CST816 是否初始化失败。

### Q2：Wi-Fi 连不上？

检查 `menuconfig` 或源码中的 SSID/密码；先用 `wifi_scan` 确认能扫到 AP；Korvo 的 Wi-Fi 在 S3 芯片内置，无需等待协处理器。

### Q3：摄像头无画面？

确认 DVP 24P 排线插紧；检查传感器型号（OV2640/OV3660 等）与例程配置一致；分辨率越高占用 PSRAM 越多。

### Q4：编译报错找不到组件？

```powershell
idf.py fullclean
idf.py set-target esp32s3
idf.py build
```

确认网络可访问 Component Registry。

### Q5：和官方 ESP32-S3-Korvo-2 例程兼容吗？

本板硬件与官方 Korvo 2 V3 引脚兼容，可直接替换使用官方大部分例程；本仓库例程针对酷世 DIY 板做了统一 UI 和 BSP 封装，学习曲线更平缓。

---

## 九、相关文档

| 文档 | 说明 |
|------|------|
| [热门例程详解](./热门例程详解.md) | Korvo / P4 分目录的单例程教程 |
| [Korvo与P4C5例程对照表](./Korvo与P4C5例程对照表.md) | 查类似工程在哪块板 |
| [例程开发流程与公共组件](./例程开发流程与公共组件.md) | BSP 与编译流程 |

---

## 八、与通用教程的对应关系

| 本仓库通用教程 | Korvo 对应例程 |
|----------------|----------------|
| [GPIO使用教程](./通用-gpio.md) | `01.basic.gpio`、`01.basic.blink` |
| [ADC使用教程](./通用-adc.md) | `01.basic.adc_oneshot`、`01.basic.adc_continuous` |
| [I2C通信教程](./通用-i2c.md) | `01.basic.i2c` |
| [WiFi使用教程](./通用-wifi.md) | `02.beginner.wifi_*` |
| [创建组件](./通用-创建组件.md) | `components/` 下各公共组件 |
| [TCA9554 I/O扩展器](./通用-tca9554.md) | 背光控制相关例程 |

---

**学完整套路线后，你应该能**：独立创建 ESP-IDF 工程；驱动 Korvo 的 LCD/触摸/音频/摄像头；用 LVGL 做 UI；做联网和 USB 应用；用 BSP 或 Board Manager 组织板级代码。

酷世DIY · Kevincoooool
