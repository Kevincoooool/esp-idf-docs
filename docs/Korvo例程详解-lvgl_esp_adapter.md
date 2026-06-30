# Korvo · LVGL esp_lvgl_adapter

> **适用开发板**：Korvo S3 · **工程** `03.development.lvgl_esp_adapter` · **target** `esp32s3`  
> **前置**：[SPI 屏触摸](./Korvo例程详解-spi_lcd_touch.md)  
> **内容**：`esp_lvgl_adapter` + BSP 跑 **`lv_demo_benchmark`**

在 Korvo **240×280 SPI 屏**上跑 LVGL 官方 benchmark，验证 adapter 与 `ksdiy_korvo_bsp` 集成。P4 MIPI 版见 [P4C5 lvgl_esp_adapter](./P4C5例程详解-lvgl_esp_adapter.md)。

---

## 1. 编译烧录

```powershell
cd Korvo_Firmware\03.development.lvgl_esp_adapter
idf.py set-target esp32s3
idf.py build flash monitor
```

---

## 2. 运行现象

屏幕循环 LVGL benchmark 场景，观察 FPS 与流畅度；串口可有性能 log。

---

## 3. 架构

```text
LVGL 9 → esp_lvgl_adapter → ST7789 SPI
CST816S → indev
```

产品工程应复用 BSP + adapter，勿重复 [spi_lcd_touch](./Korvo例程详解-spi_lcd_touch.md) 里手写 port。

---

## 4. 关键文件

- `main/app_main.c`  
- `components/ksdiy_korvo_bsp`  
- `esp_lvgl_adapter` 组件  

---

## 5. 常见问题

| 现象 | 处理 |
|------|------|
| 白屏 | SPI/背光 TCA9554 |
| FPS 低 | PSRAM；LVGL 缓冲配置 |
| 触摸偏 | 旋转与 adapter 一致 |

---

## 6. 下一步

[MP3](./Korvo例程详解-mp3_player.md) · `03.development.lvgl_v9`

酷世DIY · Kevincoooool
