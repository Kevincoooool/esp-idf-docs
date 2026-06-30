# P4C5 · 人脸识别（ESP-DL）

> **适用开发板**：P4C5 · **工程** `04.advanced.face_recognition` · **target** `esp32p4`  
> **硬件**：MIPI CSI **SC2336**（必选）· **P4 AI 加速器**  
> **特点**：纯本地推理，**无需联网**

基于 **ESP-DL** 在 P4 上跑多种视觉 AI，可按 menuconfig **组合启用**：人脸检测、人脸识别、猫脸、颜色、运动检测。摄像头固定 **QVGA 320×240** 以保证帧率。

---

## 1. 这个例子在干嘛

1. CSI 采集 RGB565 帧  
2. 多个 AI 模块通过 **FreeRTOS 队列** 串成流水线：采集 → 检测 → 识别 → … → 显示  
3. `lvgl_display_task` 消费最终帧，在 LVGL 上 **叠加检测框/文字**  
4. 人脸识别需 **SPIFFS** 存特征库（首次可能格式化分区）  

---

## 2. 编译烧录

```powershell
cd P4_C5_4.3_Firmware\04.advanced.face_recognition
idf.py set-target esp32p4
idf.py menuconfig
idf.py build flash monitor
```

### menuconfig 关键

**Face Recognition AI Features**（名称以 Kconfig 为准）勾选需要的功能，例如：

- Human Face Detection  
- Human Face Recognition  
- Cat Face / Color / Motion  

也可改源码 `main/app_ai_config.h` 的 **`APP_AI_ENABLED_FEATURES`** 掩码（与 menuconfig 对应）。

**Component Manager / ESP-DL**：确保模型组件已拉取，sdkconfig 中 PSRAM、AI 加速相关项开启。

---

## 3. 运行现象

- 屏幕实时预览（约 320×240 区域）  
- **人脸检测**：绿色矩形框  
- **人脸识别**：框上方显示已注册姓名  
- **猫脸/颜色/运动**：对应框或标记  
- 串口：坐标、置信度等  

---

## 4. 关键文件

| 文件 | 作用 |
|------|------|
| `main/app_main.cpp` | 注册流水线、显示任务 |
| `main/app_ai_config.h` | 功能开关 |
| `main/who_human_face_detection.cpp` | 人脸检测 |
| `main/who_human_face_recognition.cpp` | 特征提取 + 比对 |
| `main/who_cat_face_detection.cpp` 等 | 其它检测 |
| `main/who_camera.c` | 摄像头采集 |
| `main/camera_lvgl_view.c` | LVGL 叠加框 |
| `main/event_logic.cpp` | 事件逻辑 |

---

## 5. 流水线架构（核心）

```text
register_selected_ai()
  摄像头 → ai_frame_queue → 模块1 → 模块2 → … → display_frame_queue
                                                          ↓
                                              lvgl_display_task → 屏幕
```

每个 `who_*` 模块：**从队列取帧 → 推理/画框 → 送入下一队列**。  
学架构重点看 `app_main.cpp` 的 **`register_selected_ai()`**。

---

## 7. 人脸识别使用提示

- 首次需 **注册人脸**（工程内通常有按键/流程，见 README 或 UI）  
- 特征存 SPIFFS，换分区或 erase 会丢失库  
- 光照、角度影响识别率；QVGA 下距离 0.5～1.5 m 较合适  

---

## 8. 常见问题

| 现象 | 处理 |
|------|------|
| 编译缺模型 | `idf.py reconfigure`；Component Manager 网络 |
| 无框 | 功能是否在 menuconfig 启用；摄像头是否正常 |
| 识别全 Unknown | 是否注册；特征库路径 |
| 帧率很低 | 少开几个 AI 模块；确认用硬件加速 |
| Korvo 工程不能直接烧 P4 | 模型与 CSI 驱动不同 |

---

## 9. 动手改造

1. menuconfig 只开「检测」不开「识别」，先看框  
2. 检测到人脸时 GPIO 亮灯  
3. 串口打印 bbox 坐标做上位机  
4. 更多模型：`05.test.espdl_*`  

Korvo DVP 版 → [Korvo face_recognition](./Korvo例程详解-face_recognition.md)。

---

## 10. 下一步

[qrcode_detection](./P4C5例程详解-qrcode_detection.md) → [camera_lvgl_display](./P4C5例程详解-camera_lvgl_display.md)

酷世DIY · Kevincoooool
