# Korvo · 二维码识别

> **适用开发板**：Korvo S3 · **工程** `04.advanced.qrcode_detection` · **target** `esp32s3`  
> **库**：`esp_code_scanner` · **摄像头**：DVP  

DVP 取帧 → 二维码/条码解码 → LVGL 显示内容与预览。逻辑与 [P4C5 qrcode](./P4C5例程详解-qrcode_detection.md) 相同，硬件为 **单路 DVP** 非 MIPI/UVC。

---

## 1. 编译烧录

```powershell
cd Korvo_Firmware\04.advanced.qrcode_detection
idf.py set-target esp32s3
idf.py build flash monitor
```

用手机屏幕展示 QR 码，距离 **10～30 cm**，清晰占画面一定比例。

---

## 2. 运行现象

- 上方/主区域：摄像头预览  
- 识别成功：label 显示字符串，串口同步打印  

支持 QR Code、Data Matrix、PDF417 等（以 scanner 配置为准）。

---

## 3. 关键文件

| 文件 | 作用 |
|------|------|
| `main/app_main.c` | 入口 |
| `main/page_cam.c` | `Cam_Task` + scanner |
| `main/app_camera.c` | DVP init |

---

## 4. 核心循环

`Cam_Task`：取 RGB565 → `esp_code_scanner_scan` → 字节序转换 → 更新 LVGL。

---

## 5. 常见问题

| 现象 | 处理 |
|------|------|
| 扫不出 | 距离/光线/模糊 |
| 花屏 | RGB565 swap |
| 黑屏 | 同 camera_lvgl_display 排查 |

---

## 6. 下一步

[face_recognition](./Korvo例程详解-face_recognition.md)

酷世DIY · Kevincoooool
