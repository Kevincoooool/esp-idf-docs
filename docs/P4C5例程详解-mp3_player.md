# P4C5 · MP3 播放器

> **适用开发板**：P4C5 · **工程** `03.development.mp3_player` · **target** `esp32p4`  
> **前置**：[MIPI 屏 + LVGL](./P4C5例程详解-mipi_lcd_touch.md)  
> **技术栈**：Helix 软解 + SPIFFS + ES8311 + LVGL 圆盘 UI + **FFT 频谱**

不依赖 ESP-ADF，纯 ESP-IDF 实现完整播放器：扫描 SPIFFS 内 MP3、解码输出、触摸切歌/音量、底部频谱条随音乐跳动。

---

## 1. 这个例子在干嘛

1. 挂载 **SPIFFS**，递归扫描 `.mp3`  
2. 初始化 **LVGL** 暗色圆盘播放器 UI（曲目下拉、播放/暂停、切歌、音量滑块）  
3. 音频任务用 **Helix** 逐帧解码 → 立体声混单声道 → `esp_codec_dev_write` → ES8311  
4. 解码 PCM 送 **FFT**，实时更新频谱画布  
5. 播放控制经 **FreeRTOS 队列** 异步下发，UI 更新用 `lv_async_call` 保证线程安全  

---

## 2. 准备 MP3 文件

MP3 需打进 SPIFFS 分区：

1. 把 `.mp3` 放入工程目录 **`spiffs_image/`**  
2. 编译时自动打包进 `storage` 分区  

```powershell
# 示例
copy D:\Music\test.mp3 spiffs_image\
```

---

## 3. 编译烧录

```powershell
cd P4_C5_4.3_Firmware\03.development.mp3_player
idf.py set-target esp32p4
idf.py build flash monitor
```

接扬声器（板载 PA，GPIO48 使能）。本工程 **只初始化 DAC 播放路径**，不初始化 ES7210 ADC，避免 16 kHz 录音与 44.1 kHz MP3 **双工采样率冲突**（README 重要提示）。

---

## 4. 运行现象

- 金色圆盘 UI、曲名滚动、下拉选曲  
- 中央播放/暂停，左右切歌  
- 音量滑块 0～90  
- 底部 **频谱条** 随节拍跳动  
- 列表播完循环  

---

## 5. 关键文件

| 文件 | 作用 |
|------|------|
| `main/app_main.c` | NVS、SPIFFS、LVGL、启动播放器 |
| `main/audio.c` | 扫描、ID3、Helix 解码、播放任务、队列控制 |
| `main/ui_audio.c` | 圆盘 UI、控件回调 |
| `main/fft_convert.c` / `spectrum_display.c` | 频谱计算与绘制 |

---

## 6. 核心流程

```text
app_main
  → spiffs_mount + scan_mp3
  → ksdiy_lvgl_port_init + ui_create_disc_deck()
  → xTaskCreate(audio_task)

audio_task 循环:
  → 打开当前 mp3 → 解析 ID3 → 找同步字
  → helix 解码帧 → 采样率变化时重配 codec
  → write ES8311 + 送 FFT
  → 队列事件: PLAY/PAUSE/NEXT/PREV/SELECT
```

### LVGL 线程安全

修改 UI 必须在 LVGL 任务上下文。音频任务里用 **`lv_async_call`** 更新曲名、按钮状态，不要直接 `lv_label_set_text`。

### 锁

若与别的任务共享 LVGL，用 BSP 提供的 **`ksdiy_lvgl_lock` / unlock**。

---

## 7. 常见问题

| 现象 | 处理 |
|------|------|
| 无声音 | 扬声器/PA；音量滑块；SPIFFS 是否有 mp3 |
| 编译后仍无曲目 | 是否放入 `spiffs_image/` 并重新 build（会重打 SPIFFS） |
| 卡顿/杂音 | MP3 码率过高；PSRAM 是否开启 |
| 频谱不动 | 是否在播放；看 `fft_convert` 是否被调用 |

---

## 8. 动手改造

1. 改 `UI_COLOR_GOLD` 等换主题色  
2. 改 `DEFAULT_VOLUME`  
3. 播放结束自动下一曲（若未实现可补逻辑）  
4. 产品级多媒体可对比 [GMF 专题](./P4C5-GMF多媒体例程.md)  

---

## 9. 下一步

- `03.development.audio_record_play` — 录音回放  
- `03.development.usb_audio_player` — USB 扬声器  
- [AVI 播放](./P4C5例程详解-avi_player.md)

Korvo 小屏 SPI 版见 [Korvo MP3](./Korvo例程详解-mp3_player.md)。

酷世DIY · Kevincoooool
