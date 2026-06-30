# Korvo · 摄像头 LVGL 预览（DVP）

> **适用开发板**：Korvo S3 · **工程** `04.advanced.camera_lvgl_display` · **target** `esp32s3`  
> **摄像头**：**DVP** 并口 sensor（非 P4 的 MIPI CSI）

单路摄像头采集，在 **LVGL** 界面里用 `lv_img` 显示实时 RGB565 预览。Korvo **无双路 CSI+UVC**，P4 双路见 [P4C5 camera_lvgl_display](./P4C5例程详解-camera_lvgl_display.md)。

---

## 1. 这个例子在干嘛

1. `app_camera.c` 初始化 DVP 与 sensor  
2. `page_cam.c` 创建 LVGL 页面与取帧任务  
3. 循环：取帧 → 格式转换 → 更新 `lv_img_dsc`  

---

## 2. 编译烧录

```powershell
cd Korvo_Firmware\04.advanced.camera_lvgl_display
idf.py set-target esp32s3
idf.py menuconfig    # 摄像头型号、引脚（若可配）
idf.py build flash monitor
```

确认 Korvo 摄像头模组/FPC 已接（验板可先跑 [korvo_board_test](./Korvo例程详解-korvo_board_test.md) Camera 页）。

---

## 3. 运行现象

- LVGL 窗口内实时画面  
- 串口：init、帧率或错误 log  

---

## 4. 关键文件

| 文件 | 作用 |
|------|------|
| `main/app_main.c` | 入口、加载 page |
| `main/app_camera.c` | DVP 驱动 |
| `main/page_cam.c` | UI + Cam_Task |

---

## 5. 技术注意

- **分辨率**：QVGA 常见，过高 SPI 屏刷新吃紧  
- **RGB565 字节序**：与 LVGL 不一致时需 swap（同 P4 qrcode 文档）  
- **帧缓冲**：优先 PSRAM  

---

## 6. 常见问题

| 现象 | 处理 |
|------|------|
| 黑屏 | FPC；I2C 检测 sensor；board_test Camera 页 |
| 花屏 | 字节序；宽高与 dsc 不匹配 |
| 卡顿 | 降分辨率；减小 LVGL 刷新区域 |

---

## 7. 下一步

- [camera_webserver](./Korvo例程详解-camera_webserver.md) — 浏览器看图  
- [face_recognition](./Korvo例程详解-face_recognition.md)  
- `04.advanced.color_tracking_lvgl`

酷世DIY · Kevincoooool
