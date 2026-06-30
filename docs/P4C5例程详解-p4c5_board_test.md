# P4C5 · 整板综合测试 p4c5_board_test

> **适用开发板**：ESP32P4-C5 4.3 寸触摸屏  
> **工程**：`P4_C5_4.3_Firmware/04.advanced.p4c5_board_test`  
> **target**：`esp32p4`  
> **用途**：出厂/买家 **约 30 分钟验板**；赛博风 NODE P4 主界面  

一块程序覆盖 MIPI 屏、触摸、CSI、UVC、IMU、音频、C5 Wi-Fi 扫描等，比逐个例程更快判断「板子是否好」。

Korvo 验板见 [korvo_board_test](./Korvo例程详解-korvo_board_test.md)。

---

## 1. 主界面有什么

- **健康度**指示与 **频谱动画**（装饰性 UI）  
- **8 个磁贴**入口，各测一项硬件/功能  

| 磁贴 | 测试内容 |
|------|----------|
| MIPI CSI | SC2336 模组预览 |
| USB UVC | USB Host + UVC 摄像头（GPIO20 升压） |
| 按键 | 板载按键响应 |
| LSM6DS3 | IMU 六轴数据 |
| 硬件检测 | 汇总 I2C/外设探测 |
| Wi-Fi 扫描 | **C5** 扫 AP 列表 |
| 语音唤醒 | ES8311/ES7210 链路（若启用） |
| 关于 | 版本/信息 |

---

## 2. 编译烧录

```powershell
cd P4_C5_4.3_Firmware\04.advanced.p4c5_board_test
idf.py set-target esp32p4
idf.py build flash monitor
```

无需 menuconfig 配 Wi-Fi（扫描页为被动扫描）。CSI/UVC 需对应硬件已接。

---

## 3. 推荐验板顺序

1. **主界面** — 屏亮、触摸能点磁贴  
2. **硬件检测** — I2C 设备是否齐（AXP、触摸、音频等）  
3. **MIPI CSI** — 有模组则应有画面  
4. **Wi-Fi 扫描** — **等 C5 就绪 10～20 s 再进**，应能看到周边 AP  
5. **UVC** — 接 USB 摄像头 + 升压，等枚举  
6. **IMU** — 晃动板子数值变化  
7. **音频** — 若 ES8311/ES7210 init 失败，页内 **红屏报错**  

---

## 4. 关键文件

| 文件 | 作用 |
|------|------|
| `main/page_*.c` | 各磁贴页面 UI 与逻辑 |
| `main/wifi_scanner.c` | C5 Wi-Fi 扫描 |
| `main/app_csi_camera.c` | CSI 预览 |
| `main/lsm6ds3.c` | IMU 驱动 |

---

## 5. 注意事项

- **Wi-Fi 扫描**依赖 C5，与 [wifi_station](./P4C5例程详解-wifi_station.md) 相同需等待协处理器  
- **CSI** 需 SC2336 FPC 接牢  
- **UVC** 需 Type-A 口 5V（GPIO20）  
- 音频失败不一定是「坏板」，有时是排线/喇叭未接 — 看红屏具体项  

---

## 6. 常见问题

| 现象 | 处理 |
|------|------|
| 扫描为空 | C5 未就绪；2.4G 环境；再进一次 |
| CSI 黑屏 | FPC；硬件检测里是否识别 sensor |
| UVC 无设备 | 升压 GPIO20；换线/摄像头 |
| 触摸无效 | 对比 [mipi_lcd_touch](./P4C5例程详解-mipi_lcd_touch.md) |

---

## 7. 验板通过后学什么

[mipi_lcd_touch](./P4C5例程详解-mipi_lcd_touch.md) → [wifi_station](./P4C5例程详解-wifi_station.md) → [P4C5 学习指南](./P4C5例程学习指南.md)

酷世DIY · Kevincoooool
