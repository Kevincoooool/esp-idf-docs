# P4C5 · LVGL esp_lvgl_adapter

> **适用开发板**：P4C5 · **工程** `03.development.lvgl_esp_adapter` · **target** `esp32p4`  
> **前置**：[MIPI 屏触摸](./P4C5例程详解-mipi_lcd_touch.md)  
> **内容**：官方 **`esp_lvgl_adapter`** + BSP 跑 **`lv_demo_benchmark`**

在 P4 MIPI 480×800 上跑 LVGL 性能基准，测 FPS、刷新率，验证 `ksdiy_p4c5_bsp` 与 adapter 集成是否正常。产品 UI 应基于此栈而非重复手写 MIPI init。

---

## 1. 这个例子在干嘛

1. BSP 初始化 MIPI ST7102 + 触摸 ST7123 + 背光 AXP  
2. `esp_lvgl_adapter` 创建 display / indev  
3. 运行 **`lv_demo_benchmark()`** 全屏 benchmark  
4. 串口/屏上可看场景切换与 FPS  

旋转专项：`03.development.lvgl_esp_adapter_rotate`。

---

## 2. 编译烧录

```powershell
cd P4_C5_4.3_Firmware\03.development.lvgl_esp_adapter
idf.py set-target esp32p4
idf.py build flash monitor
```

无需 Wi-Fi。首次建议先跑通 [mipi_lcd_touch](./P4C5例程详解-mipi_lcd_touch.md) 确认硬件。

---

## 3. 运行现象

- 屏幕循环播放 LVGL benchmark 各测试场景（填充、文字、图片等）  
- 可观察是否流畅、有无 tearing  
- 触摸部分场景可交互（以 demo 为准）  

---

## 4. 关键文件

| 文件 | 作用 |
|------|------|
| `main/app_main.c` | BSP + adapter + demo 入口 |
| `components/ksdiy_p4c5_bsp` | 板级 MIPI/触摸/PMIC |
| `esp_lvgl_adapter` | LVGL 与 LCD 桥接 |

---

## 5. 架构

```text
LVGL 9
  ↓ esp_lvgl_adapter (flush / rotation / tear avoid)
  ↓ MIPI DSI ST7102 480×800
ST7123 → adapter indev → LVGL pointer
```

与 `mipi_lcd_touch` 手写 port 对比：adapter 统一 **多缓冲、旋转、性能调优**，后续 MP3/相册/小智等工程均走 BSP+adapter。

---

## 6. 常见问题

| 现象 | 处理 |
|------|------|
| 不亮屏 | AXP/MIPI LDO；先 board_test |
| FPS 低 | PSRAM 带宽；双缓冲配置；CPU 频率 |
| 触摸偏移 | 旋转角与 adapter 旋转一致 |

---

## 7. 动手改造

1. 把 `lv_demo_benchmark` 换成 `lv_demo_widgets`  
2. 跑 `lvgl_esp_adapter_rotate` 测四向旋转  
3. 对比 `sdkconfig` 中 LVGL 内存与 `CONFIG_LV_MEM_SIZE`  

Korvo SPI 版 → [Korvo lvgl_esp_adapter](./Korvo例程详解-lvgl_esp_adapter.md)。

---

## 8. 下一步

[MP3](./P4C5例程详解-mp3_player.md) · [esp-album 相册](./P4C5例程详解-esp-album.md)

酷世DIY · Kevincoooool
