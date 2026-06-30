# P4C5 · 数字相册 esp-album

> **适用开发板**：ESP32P4-C5 · **工程** `04.advanced.esp-album-ksdiy` · **target** `esp32p4`  
> **仅 P4 仓库**；硬件 JPEG/PPA、MIPI 大屏、SD 卡 slot0 + C5 SoftAP。  
> Korvo 相册为 `03.development.photo_album`，能力与工程均不同，勿混用。

---

## 1. 这个例子在干嘛

**完整数字相册产品向 Demo**：

- TF 卡 **幻灯片** 播放 JPEG/PNG（硬件 JPEG 解码 + PPA 缩放）
- 播放 **MP4**（MJPEG 视频 + AAC 音频）
- **触摸手势**：左右滑切图、上下滑音量、单击暂停、长按设置
- **Wi-Fi SoftAP**：手机连热点 → 浏览器上传照片
- **USB MSC**：接电脑时 TF 卡变 U 盘（播放自动暂停）

Korvo 相册为 `03.development.photo_album`，能力不同。

---

## 2. 硬件现象

| 状态 | 现象 |
|------|------|
| 有 TF 卡 + 有图 | 自动幻灯片 |
| 无媒体 | 显示 "No Media" |
| SoftAP 开 | 手机连 `PhotoAlbum_WiFi`（默认）→ 浏览器 `http://192.168.4.1` 上传 |
| USB 接 PC | 资源管理器出现 U 盘盘符 |

---

## 3. TF 卡准备

1. **FAT32** 格式化
2. 目录结构：

```text
/sdcard/photos/     ← JPG、PNG
/sdcard/videos/     ← MP4（MJPEG+AAC）
```

### MP4 转码（P4 要求）

```bash
ffmpeg -i input.mp4 -vcodec mjpeg -vf scale=480:800 -r 15 \
  -acodec aac -b:a 128k output.mp4
```

**不支持 H.264 直解**，必须 MJPEG 视频轨 + AAC 音频轨。

---

## 4. 编译烧录

```powershell
cd P4_C5_4.3_Firmware\04.advanced.esp-album-ksdiy
idf.py set-target esp32p4
idf.py menuconfig
idf.py build flash monitor
```

插入 TF 卡后再上电或复位。

---

## 5. menuconfig

```
Digital Photo Album Configuration →
    WiFi File Server Configuration →
        Enable WiFi File Server      默认 y
        WiFi AP SSID                   默认 PhotoAlbum_WiFi
        WiFi AP Password               默认 12345678
    Audio Decoder / Video Sync 等

一般不需要 Example Connection Configuration（SoftAP 模式，不连路由器）
```

---

## 6. 关键文件

| 模块 | 文件 |
|------|------|
| 入口 | `main/main.c` |
| 相册核心 | `main/core/photo_album.c` |
| 图片解码 | `main/media/image_decoder.c` |
| 视频 | `main/media/video_player.c` |
| 网络 | `main/network/wifi_server.c`、`app_http_server.c` |
| USB | `main/usb/usb_msc.c` |
| UI | `main/ui/ui_manager.c` |

---

## 7. 核心架构

```text
main
  ├── SDMMC 挂载 /sdcard
  ├── ksdiy_lvgl_port_init()
  ├── photo_album_scan(photos/, videos/)
  └── ui_manager + 手势

幻灯片: image_decoder (HW JPEG) → PPA scale → LVGL 显示
视频:   video_player → MJPEG 帧 + AAC → I2S

SoftAP: C5 esp_hosted → HTTP 上传 → 写入 /sdcard/photos/
USB MSC: SD 卡切换为 U 盘模式（与播放互斥）
```

---

## 8. P4 专项说明

| 项目 | 说明 |
|------|------|
| JPEG | **硬件解码** + **PPA** 缩放到 480×800 |
| Wi-Fi AP | **C5** 提供 SoftAP |
| SD 卡 | **SDMMC slot 0**（slot 1 被 C5 SDIO 占用） |
| 显示 | MIPI DSI + ST7123 触摸 |
| 内存 | 大图为 PSRAM 缓冲 |

---

## 9. 手机上传流程

1. 板子启动相册，SoftAP 开启（默认 SSID/密码见 menuconfig）
2. 手机 Wi-Fi 连接 `PhotoAlbum_WiFi`
3. 浏览器打开 **`http://192.168.4.1`**
4. 选择 JPG/PNG 上传
5. 板子幻灯片刷新显示新图

---

## 10. 动手改造

1. 改幻灯片间隔时间
2. 上传 API 加简单密码
3. 仅 USB MSC 模式，关 SoftAP 减功耗
4. 加 MQTT：远程触发「下一张」

---

## 11. 常见问题

| 现象 | 处理 |
|------|------|
| No Media | TF 未插；无 photos 目录；非 FAT32 |
| 图片无法显示 | 格式；过大分辨率；解码 log |
| 视频无画面 | 非 MJPEG+AAC；ffmpeg 重转 |
| 手机连不上 AP | SSID/密码；仅 2.4G 手机热点干扰 |
| USB 与 SD 冲突 | MSC 模式下暂停播放属正常 |
| SD 与 C5 冲突 | 必须用 slot 0 引脚配置 |

---

## 12. 相关例程

| 例程 | 关系 |
|------|------|
| `03.development.photo_album` | Korvo 简化相册 |
| `03.development.lvgl_sdcard_jpg` | 单张 JPG |
| [avi_player](./P4C5例程详解-avi_player.md) | AVI 路线 |
| `04.advanced.mjpeg_album` | 另一套相册 UI |

酷世DIY · Kevincoooool
