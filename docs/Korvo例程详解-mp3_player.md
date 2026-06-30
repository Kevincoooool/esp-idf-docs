# Korvo · MP3 播放器

> **适用开发板**：Korvo S3 · **工程** `03.development.mp3_player` · **target** `esp32s3`  
> **前置**：[SPI 屏 + LVGL](./Korvo例程详解-spi_lcd_touch.md)

Helix MP3 软解 + SPIFFS + ES8311 + LVGL 播放器 UI，结构与 P4C5 的 `mp3_player` 同类，但屏幕为 **SPI 小屏**，无 P4 的大频谱动效（以实际 UI 为准）。

P4 圆盘+频谱版见 [P4C5 MP3](./P4C5例程详解-mp3_player.md)。

---

## 1. 这个例子在干嘛

1. 挂载 SPIFFS，扫描 `.mp3`  
2. 初始化音频 codec（ES8311）与 LVGL 播放器界面  
3. 音频任务：Helix 解码 → I2S/`esp_codec_dev_write`  
4. UI：播放/暂停、切歌、音量（见 `ui_audio.c`）  

---

## 2. 准备 MP3

将文件放入 **`spiffs_image/`**，重新 build 会把 SPIFFS 打进固件。

---

## 3. 编译烧录

```powershell
cd Korvo_Firmware\03.development.mp3_player
idf.py set-target esp32s3
idf.py build flash monitor
```

---

## 4. 关键文件

| 文件 | 作用 |
|------|------|
| `main/app_main.c` | 入口 |
| `main/audio.c` | 解码与播放控制 |
| `main/ui_audio.c` | LVGL UI |
| `main/app_speech.c` | codec 相关 |

---

## 5. 核心流程

```text
spiffs_mount → scan mp3 → ui init → audio_task
  → ID3 + helix 解码 → 采样率变化重配 codec → 写 ES8311
UI 事件 → 队列 → audio_task
```

UI 更新用 **`lv_async_call`**，勿在音频任务里直接调 LVGL API。

---

## 6. 常见问题

| 现象 | 处理 |
|------|------|
| 无曲目 | `spiffs_image/` 是否有 mp3 并 rebuild |
| 无声音 | 喇叭/PA；TCA9554 背光与音频无关但需供电正常 |
| 杂音 | 码率过高；PSRAM |

---

## 7. 动手改造

1. 改默认音量  
2. 换 UI 主题色  
3. `03.development.audio_record_play` — 录音对比  

酷世DIY · Kevincoooool
