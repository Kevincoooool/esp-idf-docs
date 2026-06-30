# P4C5 · 二维码识别

> **适用开发板**：P4C5 · **工程** `04.advanced.qrcode_detection` · **target** `esp32p4`  
> **摄像头**：MIPI CSI（SC2336）或 USB UVC（由 sdkconfig 选择）

摄像头实时取帧，调用 **`esp_code_scanner`** 解码 QR/Data Matrix/PDF417 等，LVGL 上方预览、下方显示识别文本。**纯本地**，无需联网。

---

## 1. 这个例子在干嘛

1. 初始化 LVGL 与摄像头  
2. `Cam_Task` 循环：取 RGB565 帧 → `esp_code_scanner` 扫描 → 更新 `lv_img` + `lv_label`  
3. 识别成功则屏幕与串口打印内容  

---

## 2. 测试准备

- 手机生成二维码（文字、URL、Wi-Fi 配置码均可）  
- 距离摄像头 **10～30 cm**，二维码占画面足够大、清晰  
- 光线充足，避免反光  

---

## 3. 编译烧录

```powershell
cd P4_C5_4.3_Firmware\04.advanced.qrcode_detection
idf.py set-target esp32p4
idf.py menuconfig    # 摄像头类型 CSI / UVC
idf.py build flash monitor
```

UVC 模式需 **GPIO20 升压** + USB 摄像头，枚举较慢见 [camera_lvgl_display](./P4C5例程详解-camera_lvgl_display.md)。

---

## 4. 运行现象

- 上方：摄像头实时画面  
- 下方：识别到的字符串（未识别时为空或提示）  
- 串口同步打印 decode 结果  

---

## 5. 关键文件

| 文件 | 作用 |
|------|------|
| `main/app_main.c` | NVS、LVGL、`page_cam_load()` |
| `main/page_cam.c` | UI + `Cam_Task` + scanner 调用 |
| `main/app_camera.c` | 摄像头 init |

---

## 6. 核心循环（必读）

`page_cam.c` → **`Cam_Task()`**：

```text
while (1) {
  取一帧 RGB565
  esp_code_scanner_scan(...)
  字节序转换 → lv_img_dsc
  若成功 → lv_label 显示文本
}
```

### RGB565 字节序

CSI 与 LVGL 的 RGB565 **高低字节/ R-B 顺序** 可能不一致，工程内 swap 代码是常见花屏原因。

---

## 7. 常见问题

| 现象 | 处理 |
|------|------|
| 一直扫不出 | 距离/大小/对焦；提高对比度 |
| 识别慢 | 降分辨率试 QVGA；CPU 负载 |
| 花屏 | 字节序；buffer 宽高与 dsc 一致 |
| UVC 无图 | 升压、枚举等待 |

---

## 8. 动手改造

1. 识别 Wi-Fi 二维码后自动 `esp_wifi_connect`（注意安全）  
2. 识别 URL 后在 label 显示「可点击」提示  
3. 成功时蜂鸣或 LED  
4. `05.test.usb_qr_recognition` — UVC 专项  

---

## 9. 下一步

[face_recognition](./P4C5例程详解-face_recognition.md) · [camera_lvgl_display](./P4C5例程详解-camera_lvgl_display.md)

酷世DIY · Kevincoooool
