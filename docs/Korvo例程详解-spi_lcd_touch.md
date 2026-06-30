# Korvo · SPI 屏 + 触摸 + LVGL

> **适用开发板**：ESP32-S3 Korvo 2 V3  
> **工程**：`Korvo_Firmware/02.beginner.spi_lcd_touch`  
> **target**：`esp32s3`  
> **硬件**：ST7789 SPI 240×280 + CST816S I2C + TCA9554 背光  
> 相关：[CST816触摸](./通用-cst816.md)、[LVGL V9](./通用-lvgl-v9.md)

P4C5 为 MIPI 大屏，**不是本例程**，见 [P4C5 mipi_lcd_touch](./P4C5例程详解-mipi_lcd_touch.md)。

---

## 1. 这个例子在干嘛

手写完成 SPI LCD、I2C 触摸、LVGL 移植的最小闭环，后续 Korvo 图形例程的基础。

---

## 2. 硬件现象

- 默认跑 **`lv_demo_music()`**（可触摸）
- 工程内 `lvgl_demo_ui.c` 有 Meter 仪表盘，默认未调用，可自行改 `app_main` 切换

---

## 3. 编译烧录

```powershell
cd Korvo_Firmware\02.beginner.spi_lcd_touch
idf.py set-target esp32s3
idf.py build flash monitor
```

---

## 4. 引脚（本板）

| 信号 | GPIO |
|------|------|
| LCD CLK / MOSI / DC / CS | 1 / 0 / 2 / 46 |
| 触摸 SCL / SDA | 18 / 17 |
| 背光 | TCA9554 扩展器 |

---

## 5. 架构

```text
LVGL → flush → ST7789 SPI
CST816S → esp_lcd_touch → LVGL indev
esp_timer tick + 独立 LVGL 任务
```

其他 Korvo 例程多用 `ksdiy_example_display` 或 `ksdiy_korvo_bsp`，不必重复写上述 init。

---

## 6. 常见问题

- **白屏**：SPI 线序、TCA9554 背光、CS/DC  
- **触摸无效**：I2C 17/18、CST816 地址  

---

## 7. 下一步

[Korvo lvgl_esp_adapter](./Korvo例程详解-lvgl_esp_adapter.md) → `03.development.lvgl_v9` → [MP3](./Korvo例程详解-mp3_player.md)
