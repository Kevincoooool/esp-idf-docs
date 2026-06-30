# P4C5 · MIPI 屏 + 触摸 + LVGL

> **适用开发板**：ESP32P4-C5 4.3 寸触摸屏  
> **工程**：`P4_C5_4.3_Firmware/02.beginner.mipi_lcd_touch`  
> **target**：`esp32p4`  
> **硬件**：ST7102 MIPI DSI 480×800 + ST7123 触摸 + AXP2101 上电  

Korvo 为 SPI 小屏，**不是本例程**，见 [Korvo spi_lcd_touch](./Korvo例程详解-spi_lcd_touch.md)。

---

## 1. 这个例子在干嘛

MIPI DSI bring-up、触摸注册、LVGL 跑 Meter 仪表盘；左下 **ROTATE** 可 0°/90°/180°/270° 旋转。

---

## 2. 编译烧录

```powershell
cd P4_C5_4.3_Firmware\02.beginner.mipi_lcd_touch
idf.py set-target esp32p4
idf.py build flash monitor
```

---

## 3. 硬件要点

- MIPI PHY 供电：AXP2101 LDO 通道（代码内配置）
- 触摸 I2C0：SCL=8，SDA=7，INT=23
- LCD RST：GPIO22；背光 PWM：GPIO6

---

## 4. 关键文件

- `main/app_main.c` — MIPI + LVGL port
- `main/lvgl_demo_ui.c` — Meter + 旋转按钮

---

## 5. 架构

```text
LVGL 9 → esp_lvgl_adapter → ST7102 MIPI → 4.3 寸屏
ST7123 → indev
```

产品工程可只调 `ksdiy_lvgl_port_init()`（BSP），无需重复本例手写部分。

---

## 6. 常见问题

- **不亮屏**：AXP 未初始化；MIPI LDO；先跑 [p4c5_board_test](./P4C5例程详解-p4c5_board_test.md)  
- **触摸无反应**：ST7123 init 失败，查 I2C log  

---

## 7. 运行现象

- 上电 AXP 给 MIPI PHY 上电后，屏亮 **LVGL Meter 仪表盘**  
- 左下 **ROTATE** 按钮：0° / 90° / 180° / 270° 旋转 UI  
- 触摸可拖动 Meter 指针（demo 行为以 `lvgl_demo_ui.c` 为准）  

---

## 8. 常见问题（补充）

| 现象 | 处理 |
|------|------|
| 亮屏但触摸坐标反了 | 旋转角与 `esp_lvgl_adapter` 旋转一致 |
| MIPI 闪一下即黑 | LDO 时序；对比 board_test 硬件检测 |
| 编译 OOM | PSRAM、LVGL 缓冲改 defaults |

---

## 9. 下一步

[P4C5 lvgl_esp_adapter](./P4C5例程详解-lvgl_esp_adapter.md) → `lvgl_esp_adapter_rotate` → [MP3](./P4C5例程详解-mp3_player.md)

酷世DIY · Kevincoooool
