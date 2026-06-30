# Korvo · 摄像头 Web 图传

> **适用开发板**：Korvo S3 · **工程** `04.advanced.camera_webserver` · **target** `esp32s3`  
> **前置**：[Wi-Fi Station](./Korvo例程详解-wifi_station.md) + DVP 摄像头  

Wi-Fi 联网后启动 **HTTP 视频流**，浏览器访问板子 IP 即可看摄像头画面。是「远程监控 / 图传」的基础模板；**Korvo 独有** 于本套文档索引（P4 有类似能力但工程名不同，勿混 bin）。

---

## 1. 这个例子在干嘛

```text
app_main
  → Wi-Fi 连接 (app_wifi.c)
  → 摄像头 init (app_camera.c)
  → HTTP 流服务 (app_http_stream.c)
  → 浏览器 GET /stream 或同类路径 → MJPEG/ multipart 推帧
```

具体 URL 路径以 `app_http_stream.c` 为准（常见 `/stream`、根路径预览页）。

---

## 2. 编译烧录

```powershell
cd Korvo_Firmware\04.advanced.camera_webserver
idf.py set-target esp32s3
idf.py menuconfig    # Wi-Fi SSID/密码
idf.py build flash monitor
```

---

## 3. 使用步骤

1. 烧录后等 **GOT IP**，串口或屏幕看 IP  
2. 手机/PC 连 **同一 Wi-Fi**  
3. 浏览器打开 `http://<板子IP>/` 或文档/串口提示的路径  
4. 应看到实时画面（延迟与分辨率取决于编码与 Wi-Fi）  

---

## 4. 关键文件

| 文件 | 作用 |
|------|------|
| `main/app_main.c` | 总入口 |
| `main/app_wifi.c` | Station |
| `main/app_camera.c` | 取帧 |
| `main/app_http_stream.c` | HTTP server + 推流 |

---

## 5. 原理简述

- **MJPEG over HTTP**：每帧 JPEG 作为 multipart 一段发送，浏览器 `<img src="/stream">` 可显示  
- 并发连接数、帧率受 **CPU + Wi-Fi 带宽** 限制  
- 与 [HTTP Server](./Korvo例程详解-http_server.md) 相比，handler 内持续 `esp_camera_fb_get` 并写 socket  

---

## 6. 常见问题

| 现象 | 处理 |
|------|------|
| 浏览器打不开 | IP/同网段；防火墙 |
| 有页无图 | 路径是否正确；摄像头 init 是否成功 |
| 很卡 | 降分辨率/JPEG 质量；路由器距离 |
| 内存不足 | 减小 fb 数量；PSRAM |

---

## 7. 动手改造

1. 加简单 HTTP 认证  
2. 单帧快照 `/capture.jpg`  
3. 与 MQTT 联动：订阅 Topic 才开启推流  

---

## 8. 下一步

[camera_lvgl_display](./Korvo例程详解-camera_lvgl_display.md) · [HTTP Server](./Korvo例程详解-http_server.md)

酷世DIY · Kevincoooool
