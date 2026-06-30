# Korvo · AVI 视频播放

> **适用开发板**：Korvo S3 · **工程** `04.advanced.avi_player` · **target** `esp32s3`  
> **屏幕**：SPI ST7789 240×280 · **解码**：JPEG 软解/硬件（以工程配置为准）

从存储读取 **MJPEG AVI**，解码 JPEG 帧后在 LCD 上连续播放。Korvo **无 P4 的 PPA/JPEG 硬解大屏管线**，性能与分辨率低于 P4C5 版，但流程相同：容器解析 → 解码 → 刷屏。

P4 大屏硬解版见 [P4C5 AVI](./P4C5例程详解-avi_player.md)。

---

## 1. 这个例子在干嘛

1. 挂载 **SD 卡或 SPIFFS**（以工程实现为准）  
2. 打开 AVI，分离 **MJPEG 视频轨**（及可选 PCM 音频轨）  
3. `app_jpeg_dec.c` 解码 JPEG → RGB → SPI LCD 刷新  
4. 播完可循环或切下一个文件  

---

## 2. 视频文件准备

与 P4 类似，AVI 内视频轨需 **MJPEG**，音频常为 **PCM**。用 ffmpeg 转码示例（分辨率按 Korvo 小屏调整）：

```bash
ffmpeg -i input.mp4 -vcodec mjpeg -vf scale=240:280 -r 10 -q:v 20 \
  -acodec pcm_s16le -ar 16000 output.avi
```

将 `output.avi` 放入工程要求的目录（SD 卡或 `spiffs_image/`，见 README / `app_main.c`）。

---

## 3. 编译烧录

```powershell
cd Korvo_Firmware\04.advanced.avi_player
idf.py set-target esp32s3
idf.py build flash monitor
```

接扬声器可听 AVI 内音频（若工程启用 ES8311 路径）。

---

## 4. 关键文件

| 文件 | 作用 |
|------|------|
| `main/app_main.c` | 存储挂载、播放控制 |
| `main/app_jpeg_dec.c` | JPEG 解码 |
| `components/avi_player/` 或同级 | AVI 解析（若有） |

---

## 5. 与 P4C5 差异

| | Korvo | P4C5 |
|--|-------|------|
| 屏 | SPI 240×280 | MIPI 480×800 |
| 解码 | 资源更紧，宜低分辨率低帧率 | JPEG 硬解 + PPA |
| 刷屏 | SPI 带宽限制 | MIPI + 可 bypass LVGL |

**工程不可互烧**；转码分辨率需按各自屏幕改 scale。

---

## 6. 常见问题

| 现象 | 处理 |
|------|------|
| 无画面 | 是否 MJPEG；文件路径 |
| 卡顿 | 降分辨率/帧率；SPI 时钟 |
| 有画面无声音 | ES8311 初始化；AVI 是否含 PCM 轨 |

---

## 7. 下一步

- `04.advanced.avi_multi_player`  
- `04.advanced.avi_recorder`  
- [MP3](./Korvo例程详解-mp3_player.md)

酷世DIY · Kevincoooool
