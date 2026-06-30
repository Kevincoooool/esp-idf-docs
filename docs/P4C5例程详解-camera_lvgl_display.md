# P4C5 · 双路摄像头 LVGL 预览

> **适用开发板**：P4C5 · **工程** `04.advanced.camera_lvgl_display` · **target** `esp32p4`  
> **硬件**：**MIPI CSI（SC2336）** + **USB UVC** 同屏上下分屏预览  

P4 独有 **双路摄像头**：板载 MIPI 模组 + USB Host 接 UVC 摄像头，在 LVGL 里上下两块 `lv_img` 实时刷新。Korvo 为单路 DVP，见 [Korvo camera_lvgl_display](./Korvo例程详解-camera_lvgl_display.md)。

---

## 1. 这个例子在干嘛

1. 初始化 MIPI CSI 与（可选）USB UVC  
2. 创建 LVGL 页面，上下两区域分别显示两路 RGB565 帧  
3. 独立任务取帧 → 字节序/格式转换 → 更新 `lv_img_dsc`  
4. 只接一路也能跑（另一路显示占位或跳过）  

---

## 2. 硬件连接

| 路 | 说明 |
|----|------|
| MIPI CSI | 板载 SC2336 模组，FPC 接好 |
| USB UVC | Type-A 口接 UVC 摄像头；**GPIO20 拉高** 给 USB 5V 升压芯片供电 |

UVC **首次枚举** 可能需 5～15 s，属正常现象。

---

## 3. 编译烧录

```powershell
cd P4_C5_4.3_Firmware\04.advanced.camera_lvgl_display
idf.py set-target esp32p4
idf.py menuconfig
idf.py build flash monitor
```

### menuconfig 常见项

- CSI 摄像头开关、I2C 引脚（与 BSP 一致）  
- UVC 相关：USB Host 模式  
- 分辨率：影响帧率与 PSRAM 占用  

以工程 `Kconfig` / README 为准。

---

## 4. 运行现象

- 上半屏：CSI 实时画面  
- 下半屏：UVC 实时画面（若已连接）  
- 串口：初始化步骤、帧率/错误 log  

---

## 5. 关键文件

| 文件 | 作用 |
|------|------|
| `main/app_main.c` | 入口、LVGL、加载页面 |
| `main/page_cam.c` | 双路 UI、取帧任务 |
| `main/app_camera.c` | CSI 初始化 |
| UVC 相关 | USB Host 枚举与取流（见工程内文件名） |

---

## 6. 核心数据流

```text
CSI: SC2336 → esp_cam → RGB565 缓冲 → Cam_Task → lv_img 上半
UVC: USB Host → UVC 驱动 → RGB565 → Cam_Task → lv_img 下半
```

### RGB565 字节序

摄像头输出与 LVGL 期望的 **RGB/BGR 字节序** 可能不同，`page_cam.c` 里常有 swap 逻辑——改分辨率或驱动后花屏先查这里。

### 性能

双路 QVGA 在 P4 + PSRAM 下可跑；提分辨率前先估 **带宽与 cache**。

---

## 7. 常见问题

| 现象 | 处理 |
|------|------|
| CSI 黑屏 | FPC；I2C 检测 SC2336；menuconfig 是否启用 CSI |
| UVC 无画面 | GPIO20 升压；线材；等枚举完成；换 UVC 模组 |
| 仅一路正常 | 另一路未接时是否代码分支跳过 |
| 花屏/色偏 | RGB565 字节序；分辨率与 buffer 大小 |
| 卡顿 | 降分辨率；减少 LVGL 全屏刷新区域 |

---

## 8. 动手改造

1. 只保留 CSI 一路全屏  
2. 在画面上叠加 FPS 计数  
3. 加触摸按钮「拍照」存 JPEG 到 SD  
4. 专项：`05.test.usb_uvc_display` 单测 UVC  

---

## 9. 下一步

- [人脸识别](./P4C5例程详解-face_recognition.md) — CSI + ESP-DL  
- [二维码](./P4C5例程详解-qrcode_detection.md)  
- `04.advanced.avi_recorder` — 录 AVI  

酷世DIY · Kevincoooool
