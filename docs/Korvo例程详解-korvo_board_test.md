# Korvo · 整板综合测试 korvo_board_test

> **适用开发板**：ESP32-S3 Korvo 2 V3  
> **工程**：`Korvo_Firmware/04.advanced.korvo_board_test`  
> **target**：`esp32s3`  
> **用途**：收板后 **首选验板**（约 15～30 分钟）

LVGL 多页面一次测 DVP 摄像头、ADC 六键、语音/唤醒、codec 与 I2C 外设。P4C5 对应工程 UI 不同，见 [p4c5_board_test](./P4C5例程详解-p4c5_board_test.md)。

---

## 1. 启动时做什么

- **I2C 扫描**：ES8311、ES7210、TCA9554 等  
- 音频 init 失败 → **红屏报错** 并可能 **跳过 WakeNet**  
- 进入主菜单多页导航（`page_manager`）  

---

## 2. 编译烧录

```powershell
cd Korvo_Firmware\04.advanced.korvo_board_test
idf.py set-target esp32s3
idf.py build flash monitor
```

左 USB（CH9102）下载。Speech 页若需唤醒模型，确认 SPIFFS/分区含模型（以工程为准）。

---

## 3. 主菜单页面

| 页面 | 测试内容 |
|------|----------|
| **Camera** | DVP 实时预览 |
| **Button** | **GPIO5 ADC** 六键矩阵，按键有 UI/串口反馈 |
| **Speech** | 录音播放 / **WakeNet 唤醒**（codec 正常时） |
| **About** | 版本与板信息 |

---

## 4. 推荐验板顺序

1. 上电进主菜单，触摸/按键能切换页面  
2. **Camera** — 有画面、帧率在动  
3. **Button** — 六个键均有响应  
4. **Speech** — 非红屏；能录放或唤醒（视模型）  
5. 串口无反复 **panic/reboot**  

---

## 5. 关键文件

| 文件 | 作用 |
|------|------|
| `main/app_main.cpp` | 入口、硬件检测 |
| `main/page_manager.c` | 页面切换 |
| `main/page_*.c` | 各子页 |
| `main/app_camera.c` | DVP |
| `main/app_adc.c` | 六键 ADC |
| `main/app_speech.c` | 音频/唤醒 |

---

## 6. 常见问题

| 现象 | 处理 |
|------|------|
| Speech 红屏 | ES8311/ES7210 I2C；排线；对比原理图 |
| Camera 黑屏 | FPC；sensor 供电 |
| 某键无反应 | ADC 阈值；GPIO5 电路 |
| WakeNet 无反应 | 模型是否烧入；麦克风 |

---

## 7. 验收标准（简要）

- 菜单可进可退  
- Camera、Button 有明确响应  
- Speech 非致命红屏  
- 串口稳定无循环 panic  

---

## 8. 下一步

[spi_lcd_touch](./Korvo例程详解-spi_lcd_touch.md) → [wifi_station](./Korvo例程详解-wifi_station.md) → [Korvo 学习指南](./Korvo例程学习指南.md)

酷世DIY · Kevincoooool
