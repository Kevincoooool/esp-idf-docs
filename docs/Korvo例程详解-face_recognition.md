# Korvo · 人脸识别（ESP-DL）

> **适用开发板**：Korvo S3 · **工程** `04.advanced.face_recognition` · **target** `esp32s3`  
> **摄像头**：DVP · **推理**：ESP-DL 本地，无需联网  

与 P4 工程 **同名但不可互烧**：Korvo 为 DVP + S3 算力，P4 为 MIPI + AI 加速器。流程类似：采集 → ESP-DL 流水线 → LVGL 叠框。

P4 详解见 [P4C5 face_recognition](./P4C5例程详解-face_recognition.md)。

---

## 1. 功能概述

按 menuconfig / `app_ai_config.h` 启用：

- 人脸检测（框）  
- 人脸识别（SPIFFS 特征库）  
- 可选：猫脸、颜色、运动等（以工程为准）  

分辨率常为 **QVGA** 以保证帧率。

---

## 2. 编译烧录

```powershell
cd Korvo_Firmware\04.advanced.face_recognition
idf.py set-target esp32s3
idf.py menuconfig    # AI 功能、模型
idf.py build flash monitor
```

首次人脸识别可能格式化 SPIFFS 存特征库。

---

## 3. 运行现象

- 小屏实时预览 + 检测框/识别名  
- 串口：bbox、置信度  

---

## 4. 架构（与 P4 同思路）

```text
摄像头 → 队列 → who_* 模块链 → display 队列 → lvgl_display_task
```

必读：`register_selected_ai()`、各 `who_*.cpp`。

---

## 5. 关键文件

| 文件 | 作用 |
|------|------|
| `main/app_main.cpp` | 流水线注册 |
| `main/who_human_face_*.cpp` | 检测/识别 |
| `main/who_camera.c` | DVP 采集 |
| `main/camera_lvgl_view.c` | 叠框显示 |

---

## 6. Korvo 注意

- S3 **无 P4 级 AI 加速器**，模块别开太多  
- 光照与距离影响识别  
- 模型通过 Component Manager 拉取，需网络 `idf.py reconfigure`  

---

## 7. 常见问题

| 现象 | 处理 |
|------|------|
| 无框 | menuconfig 是否启用；Camera 在 board_test 是否正常 |
| 帧率低 | 只开检测；降分辨率 |
| 编译缺模型 | 组件下载失败重试 |

---

## 8. 下一步

[qrcode_detection](./Korvo例程详解-qrcode_detection.md) · [camera_lvgl_display](./Korvo例程详解-camera_lvgl_display.md)

酷世DIY · Kevincoooool
