# P4C5 · AVI 视频播放

> **适用开发板**：P4C5 · **工程** `04.advanced.avi_player` · **target** `esp32p4`  
> **硬件**：MIPI 480×800 + **JPEG 硬件解码** + **PPA 缩放** + ES8311 音频  

从 TF 卡或 SPIFFS 读 **MJPEG + PCM** 的 AVI，音画同步播放。本例 **绕过 LVGL** 直刷 MIPI，帧率高于在 LVGL 里播视频。

---

## 1. 这个例子在干嘛

1. 扫描存储里的 `.avi`（优先 **TF 卡**，无文件再 **SPIFFS**）  
2. 解析 AVI：视频轨 MJPEG 帧 → **硬件 JPEG 解码** → **PPA** 缩放到屏分辨率 → `esp_lcd_panel_draw_bitmap`  
3. 音频轨 PCM → I2S → ES8311 喇叭  
4. 当前文件播完 → 自动下一个 → 列表结束从头循环  

---

## 2. 视频文件准备

### 格式要求

| 轨道 | 编码 |
|------|------|
| 视频 | **MJPEG** |
| 音频 | **PCM S16LE**，建议 16 kHz |

不支持 H.264 直放 AVI；需 ffmpeg 转 MJPEG。

### ffmpeg 转码（480×800 竖屏）

```bash
ffmpeg -i input.mp4 -vcodec mjpeg -vf scale=480:800 -r 15 -q:v 20 \
  -acodec pcm_s16le -ar 16000 output.avi
```

### 存放位置

- **TF 卡（推荐）**：FAT32，任意目录（程序 **递归扫描** 所有 `.avi`）  
- **SPIFFS**：空间有限，需自定义分区并把 avi 打进 `storage` 分区  

---

## 3. 编译烧录

```powershell
cd P4_C5_4.3_Firmware\04.advanced.avi_player
idf.py set-target esp32p4
idf.py build flash monitor
```

插入 TF 卡后再复位。无卡则依赖 SPIFFS 内是否有 avi。

---

## 4. 运行现象

- 全屏视频，串口打印存储类型（SD/SPIFFS）和播放列表  
- 有音频轨则喇叭同步出声  
- 文件结束自动切下一个  

---

## 5. 关键文件

| 文件 | 作用 |
|------|------|
| `main/app_main.c` | 入口、回调注册、扫描 avi |
| `main/app_jpeg_dec.c` | **JPEG 硬解 + PPA 缩放**（性能核心） |
| `main/app_sdcard.c` | TF 挂载 |
| `main/app_speech.c` | ES8311 |
| `components/avi_player/` | AVI 容器解析、播控 |

---

## 6. 核心数据通路

```text
AVI 文件
  → avi_player 拆帧 (00dc=MJPEG, 01wb=PCM)
  → video_cb: jpeg_decoder_process → PPA scale → draw_bitmap
  → audio_cb: esp_codec_dev_write → ES8311
  → avi_play_end_cb: 播放下一个文件
```

### 为何用 PPA

P4 的 **PPA（Pixel Processing Accelerator）** 在硬件里做缩放/旋转，CPU 不算像素。`app_jpeg_dec.c` 里 `scale_to_screen()` 是必读。

### 内存

解码缓冲常用 **PSRAM**，且需 **DMA/cache 对齐**（`heap_caps_aligned_alloc`），否则硬解或刷屏异常。

---

## 7. menuconfig / 配置

多数在 `sdkconfig.defaults` 与工程 README；CSI 无关。关注：

- PSRAM 开启  
- 分区表含足够 **app + storage**（若用 SPIFFS）  
- SDMMC slot **0**（slot 1 给 C5）  

---

## 8. 常见问题

| 现象 | 处理 |
|------|------|
| 有声音无画面 | 是否 MJPEG 轨；ffmpeg 重转 |
| 卡顿 | 分辨率/帧率过高；增大 `buffer_size`；换小一点的 avi |
| TF 扫不到文件 | FAT32；`.avi` 扩展名；接触 |
| 颜色异常 | RGB565 字节序；看 `app_jpeg_dec` 输出格式 |
| SPIFFS 空间不足 | 改用 TF 或缩小 avi |

---

## 9. 动手改造

1. 改 `on_audio_set_clock_cb` 里音量（默认约 80）  
2. 改 `avi_player_config_t.buffer_size` 观察流畅度  
3. 用 ffmpeg 试 320×480、640×480，看 PPA 缩放效果  
4. 在画面上叠加 FPS（在 `video_cb` 里计数）  

---

## 10. 下一步

- `04.advanced.avi_multi_player` — 简化轮播  
- `04.advanced.avi_recorder` — 录制 AVI  
- `04.advanced.mjpeg_album` — 相册 + 视频 UI  

酷世DIY · Kevincoooool
