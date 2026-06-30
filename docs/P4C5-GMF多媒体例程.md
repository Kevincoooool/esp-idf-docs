# P4C5 GMF 多媒体例程（06.gmf_examples）

> 仓库路径：`P4_C5_4.3_Firmware/06.gmf_examples/`  
> 共 **24** 个子工程，基于乐鑫 **esp-gmf（Generic Media Framework）** 与 **ESP Board Manager**，适配 **P4C5** 板卡 `p4_c5_4_3`。

与手写 Helix/LVGL 的 `03.development.mp3_player` 等是**两条并行学习线**：GMF 适合产品级多媒体管道，手写例程适合理解底层原理。

---

## 一、GMF 是什么

**GMF** 是乐鑫通用媒体框架，用**管道（Pipeline）** 连接音源 → 解码 → 音效 → 输出，或 采集 → 编码 → 存储/网络。

```text
[io_embed_flash] → [decoder] → [codec_dev] → 喇叭
[io_sdcard]      → [decoder] → [audio_effects] → [codec_dev]
```

Korvo 曾是 esp-gmf 上游默认板；本仓库将板卡替换为 **P4C5**，并增加 Wi-Fi 相关示例的 **esp_hosted + esp_wifi_remote** 依赖。

---

## 二、目录结构

```text
06.gmf_examples/
├── EXAMPLES.txt              示例索引
├── create_examples.ps1         从 esp-gmf 重新生成
├── build_all.ps1               批量编译
├── _template/                  公共模板
│   ├── build.ps1               set-target + bmgr + build
│   ├── sdkconfig.defaults
│   └── partitions*.csv
└── <24 个示例子目录>/
    ├── main/
    ├── README.md / README_CN.md
    └── build.ps1
```

---

## 三、构建方法（与 01～05 不同）

### 标准步骤

```powershell
cd P4_C5_4.3_Firmware\06.gmf_examples\pipeline_play_embed_music

# 方式一：使用模板脚本
..\_template\build.ps1 build flash monitor

# 方式二：手动
idf.py set-target esp32p4
idf.py bmgr --customer-path ..\..\components -b p4_c5_4_3
idf.py build flash monitor
```

**关键点**：

1. target 必须是 **esp32p4**
2. 需先 **`idf.py bmgr`** 生成 Board Manager 代码到 `components/gen_bmgr_codes`
3. 板卡名 **`p4_c5_4_3`**（不是 Korvo 的 `ksdiy_korvo_s3`）
4. 需 Wi-Fi 的示例会自动拉 **esp_hosted**

---

## 四、示例分类（24 个）

### 4.1 基础 Pipeline（11 个）

| 目录 | 功能 |
|------|------|
| `pipeline_play_embed_music` | Flash 内嵌 MP3，最简单入门 |
| `pipeline_play_sdcard_music` | SD 卡音乐播放 |
| `pipeline_play_http_music` | HTTP 流式音乐（需 Wi-Fi） |
| `pipeline_play_multi_source_music` | 多音源切换 |
| `pipeline_loop_play_no_gap` | 无缝循环播放 |
| `pipeline_record_sdcard` | 录音到 SD |
| `pipeline_record_http` | 录音 HTTP 上传 |
| `pipeline_record_audio_muxer` | 音频封装录制 |
| `pipeline_http_download_to_sdcard` | HTTP 下载到 SD |
| `pipeline_audio_effects` | 音效处理链 |
| `pipeline_howl` | 啸叫抑制演示 |

**推荐第一个**：`pipeline_play_embed_music` — 不依赖 TF 卡/Wi-Fi，最容易跑通。

### 4.2 高级示例（13 个）

| 目录 | 功能 |
|------|------|
| `audio_capture` | esp_capture 音频采集 |
| `video_capture` | esp_capture 视频采集 |
| `video_render` | esp_video_render 渲染 |
| `video_render_player` | 视频渲染 + 播放 |
| `audio_render` | PCM 混音输出 |
| `simple_piano` | 简易钢琴 |
| `aec_rec` | AEC 回声消除录音（需 SR 分区） |
| `wwe` | Wake Word Engine 唤醒 |
| `audio_player` | ESP Player 高级音频（多格式） |
| `video_player` | ESP Player 视频播放 |
| `asrc_demo` | 异步采样率转换 ASRC |
| `fft_spectrum_print` | GMF FFT 频谱串口打印 |
| `dual_eyes` | 双眼动画（**本板无 GPIO 扩展，仅参考**） |

---

## 五、与手写例程对照

| 需求 | 手写例程（03/04 层） | GMF 示例 |
|------|----------------------|----------|
| MP3 播放 | `03.development.mp3_player` (Helix) | `pipeline_play_sdcard_music` / `audio_player` |
| SD 录音 | `audio_record_sdcard` | `pipeline_record_sdcard` |
| HTTP 音乐 | 需自己拼 | `pipeline_play_http_music` |
| AVI 视频 | `04.advanced.avi_player` | `video_player` |
| 频谱 | `audio_fft` / mp3_player 内 FFT | `fft_spectrum_print` |
| 唤醒词 | `05.test.skainet_*` | `wwe` |

| 对比项 | 手写 Helix/LVGL | GMF |
|--------|-----------------|-----|
| 学习曲线 | 低，代码直观 | 高，管道抽象 |
| 灵活性 | 完全自控 | 标准化、易扩展 |
| 适合 | 入门、小项目 | 产品、乐鑫生态 |
| BSP | `ksdiy_p4c5_bsp` | `esp_board_manager` + bmgr |

---

## 六、推荐学习路线

```text
1. 先跑通 pipeline_play_embed_music（理解 GMF 最小管道）
2. pipeline_play_sdcard_music（SD 音源）
3. pipeline_play_http_music（需 Wi-Fi + C5 就绪）
4. audio_player / video_player（ESP Player 高级 API）
5. aec_rec / wwe（与语音识别结合）
```

**前置条件**：

- 已完成 [P4C5例程学习指南](./P4C5例程学习指南.md) 验板
- 理解 [例程开发流程与公共组件](./例程开发流程与公共组件.md) 中的 Board Manager 概念
- Wi-Fi 示例前先跑 `05.test.wifi_scan_c5`

---

## 七、分区与配置

GMF 示例常用 `_template` 下多套分区：

| 文件 | 用途 |
|------|------|
| `partitions.csv` | 默认 |
| `partitions_large.csv` | 大 app |
| `partitions_sr.csv` | 含语音识别模型分区（wwe/aec_rec） |

`sdkconfig.defaults.large_app` 用于大体积示例。具体见各子目录 README_CN.md。

---

## 八、从 Korvo GMF 迁移到 P4C5

若你曾用乐鑫官方 Korvo 跑 esp-gmf：

1. 板卡从 `korvo` / `ksdiy_korvo_s3` 改为 **`p4_c5_4_3`**
2. 重新 `idf.py bmgr`
3. Wi-Fi 示例增加 **esp_hosted** 等待逻辑
4. 显示/音频 handle 通过 Board Manager 设备名获取，引脚由 YAML 定义
5. 视频类示例可启用 P4 **硬件 JPEG/H.264**

---

## 九、常见问题

| 现象 | 处理 |
|------|------|
| bmgr 失败 | 确认 `-b p4_c5_4_3` 与 components 路径 |
| 无声音 | Board Manager codec handle；对比 `codec_i2s` 测试 |
| HTTP 播放失败 | C5 Wi-Fi 未就绪；URL 可访问性 |
| 编译 OOM | 使用 `partitions_large.csv` / large_app sdkconfig |
| dual_eyes 不可用 | 预期行为，本板无双眼 GPIO 扩展 |

---

## 十、相关文档

- [P4C5 mp3_player](./P4C5例程详解-mp3_player.md) — Helix 手写路线
- [P4C5 avi_player](./P4C5例程详解-avi_player.md) — 手写 AVI 路线
- [P4C5专项测试例程](./P4C5专项测试例程.md)
- [Korvo与P4C5例程对照表](./Korvo与P4C5例程对照表.md)
- 乐鑫官方：[esp-gmf 文档](https://docs.espressif.com/projects/esp-gmf/)

酷世DIY · Kevincoooool
