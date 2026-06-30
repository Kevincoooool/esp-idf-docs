# P4C5 · OTA 空中升级（HTTP + HFS）

> **适用开发板**：ESP32P4-C5 4.3 寸触摸屏  
> **工程路径**：`P4_C5_4.3_Firmware/02.beginner.ota`  
> **芯片 target**：`esp32p4`  
> **推荐 IDF**：5.5.3  
> **前置例程**：[Wi-Fi Station](./P4C5例程详解-wifi_station.md) 能连上路由器  

本例演示 **HTTP OTA**：开发板连 Wi-Fi 后，从局域网 PC 上的 **HFS（Http File Server）** 下载新固件 `.bin`，写入备用 OTA 分区，校验通过后自动重启。屏幕带 **LVGL 仪表盘**（版本号、进度条、当前分区）。

---

## 1. 这个例子在干嘛

产品量产后很少用 USB 给每一台设备烧录，而是用 **OTA** 远程或局域网推送新固件。本工程把流程拆成你能看见的几步：

1. 启动时打印 **bootloader / 当前 app 的 SHA-256**（串口对照是否真升级了）
2. 连接 Wi-Fi（经 **C5 协处理器**，需先等 C5 就绪）
3. 从 menuconfig 里配置的 **URL** 下载固件
4. 写入 **非当前运行** 的 OTA 分区（`ota_0` ↔ `ota_1` 交替）
5. 成功后 `esp_restart()`，新固件运行

默认走 **HTTP + HFS**，方便家里没有 HTTPS 服务器时做本地实验；也支持 menuconfig 切 **HTTPS** 模式。

---

## 2. 分区表（必须先理解）

`partitions.csv`：

```text
nvs, otadata, phy_init, ota_0 (4MB), ota_1 (4MB)
```

- **没有 factory 单分区**：app 只在 `ota_0` / `ota_1` 之间切换  
- 若你以前烧过 **旧版 factory 分区** 的别的工程，必须 **`idf.py erase-flash`** 再烧本例，否则分区对不上会起不来  

串口与屏幕会显示当前运行在 **`ota_0` 还是 `ota_1`**。

---

## 3. 第一次完整操作流程（HFS 测试，推荐）

### 3.1 PC 端：安装并配置 HFS

1. 下载 [HFS (Http File Server)](https://www.rejetto.com/hfs/) for Windows  
2. 建文件夹，例如 `D:\ota_files\`  
3. 编译本工程后复制固件：

```powershell
cd F:\Github\ESP32P4_KSDIY\P4_C5_4.3_Firmware\02.beginner.ota
idf.py set-target esp32p4
idf.py build
copy build\simple_ota.bin D:\ota_files\simple_ota.bin
```

4. 把 `D:\ota_files\` **拖进 HFS 窗口** 设为共享目录  
5. HFS 菜单 **Server → Port** 设为 `8080`（或自定，与 URL 一致）  
6. 看 HFS 显示的 **本机 IP**（如 `192.168.5.10`）  
7. 电脑浏览器访问验证：

```text
http://192.168.5.10:8080/simple_ota.bin
```

应能下载 bin。**PC 与板子须在同一局域网（同一路由器）**。

### 3.2 menuconfig

```powershell
idf.py menuconfig
```

**① Example Connection Configuration**

| 项 | 填写 |
|----|------|
| WiFi SSID | 路由器名称 |
| WiFi Password | 密码 |

**② Example Configuration**

| 项 | 填写 |
|----|------|
| OTA download transport | `HTTP - HFS local test` |
| Firmware version | `1.0.0`（屏幕右上角，测试升级可改 `1.0.1`） |
| firmware upgrade url endpoint | `http://192.168.5.10:8080/simple_ota.bin`（改成你的 PC IP） |

**③ Component config → ESP HTTPS OTA**

| 项 | 必须 |
|----|------|
| Allow HTTP for OTA | **勾选**（HFS 是 http 不是 https） |

### 3.3 首次烧录到板子

```powershell
idf.py erase-flash
idf.py build flash monitor
```

`erase-flash` 在**第一次**或**改过 partitions.csv** 后必做。  
烧录后屏幕应显示版本 `1.0.0`、Running Partition、`Download URL` 等。

### 3.4 验证 OTA 升级（关键：不要 USB 重烧 app）

1. menuconfig 把 **Firmware version** 改为 `1.0.1`  
2. 重新编译并 **覆盖 HFS 上的文件**：

```powershell
idf.py build
copy build\simple_ota.bin D:\ota_files\simple_ota.bin
```

3. 按板子 **RESET** 或重新上电（**不要** `idf.py flash`）  
4. 等 Wi-Fi 连上 → 屏幕进度条走动 → 自动重启  
5. 重启后屏幕版本变为 `1.0.1`；串口 **SHA-256 for current firmware** 与升级前不同即成功  

---

## 4. 屏幕 UI 说明

| 区域 | 内容 |
|------|------|
| 顶栏 | HTTP OTA / HFS Test / **固件版本号** |
| Wi-Fi | 连接状态 + SSID |
| Firmware OTA | downloading / upgrade OK / 错误码 |
| Running Partition | `ota_0` 或 `ota_1` |
| Download URL | menuconfig 中的完整 URL |
| Download Progress | 进度条 + 百分比 + 已下载 KB |
| Activity | 当前步骤文字（Connecting… / Downloading…） |

---

## 5. 关键文件

| 文件 | 作用 |
|------|------|
| `main/simple_ota_example.c` | `app_main`、OTA 任务、`esp_https_ota()`、HTTP 事件更新 UI |
| `main/ota_ui.c` / `ota_ui.h` | LVGL 仪表盘 |
| `main/Kconfig.projbuild` | 版本号、URL、HTTP/HTTPS 选择 |
| `partitions.csv` | 双 OTA 4MB |
| `server_certs/ca_cert.pem` | HTTPS 模式用的 CA |

---

## 6. 核心代码流程

```text
app_main()
  ├── ota_ui_init(版本, URL)
  ├── nvs_flash_init()
  ├── get_sha256_of_partitions()      // 串口打印 hash
  ├── show_running_partition()        // 屏幕显示 ota_0/ota_1
  ├── example_connect()               // Wi-Fi
  ├── esp_wifi_set_ps(WIFI_PS_NONE)   // 关省电，下载更稳
  └── xTaskCreate(simple_ota_example_task)

simple_ota_example_task()
  ├── esp_http_client_config（url + _http_event_handler）
  ├── esp_https_ota(&ota_config)      // HTTP 模式也走此 API
  └── 成功 → esp_restart()
```

### HTTP 事件与进度条

`_http_event_handler` 里：

- `HTTP_EVENT_ON_CONNECTED` → 取 `Content-Length` 作为总大小  
- `HTTP_EVENT_ON_DATA` → 累加 `data_len`，调用 `ota_ui_set_progress(done, total)`  
- 失败 → `ota_ui_set_ota(ERR, …)`  

改进度逻辑或加日志，从这里下手。

---

## 7. HTTPS 模式（可选）

menuconfig → **HTTPS - certificate required**：

- 可开 **Certificate Bundle** 访问公网 HTTPS  
- 或用自己的 `server_certs/ca_cert.pem`  
- URL 改为 `https://...`  
- **不必**勾选 Allow HTTP for OTA  

---

## 8. P4C5 特别注意

- Wi-Fi 走 **C5**，上电后等 10～20 s 再判断「连不上网」  
- 仅 **2.4GHz** 路由器  
- 下载期间已 `esp_wifi_set_ps(WIFI_PS_NONE)`，仍失败先查 HFS/防火墙  
- OTA 测试阶段用 **RESET 触发**，避免每次 `flash` 把新固件又写进当前分区导致「看不出升级」  

---

## 9. 常见问题

| 现象 | 处理 |
|------|------|
| Wi-Fi 连不上 | menuconfig SSID/密码；C5 是否就绪；2.4G |
| 浏览器能下 bin，板子 OTA 失败 | URL 是否 `http://`；Allow HTTP for OTA 是否勾选；文件名是否 `simple_ota.bin` |
| 进度条不动 | PC 防火墙放行 8080；HFS 是否仍在运行 |
| 升级后版本不变 | HFS 上 bin 是否已替换为新编译；是否误用 `flash` 而非 RESET |
| 启动 panic / 分区错误 | `idf.py erase-flash` 后重烧 |
| SHA 不变 | 下载的仍是旧 bin（未 copy 新文件到 HFS） |

---

## 10. 动手改造

1. OTA 成功后把新版本号通过 [MQTT](./P4C5例程详解-mqtt_tcp.md) 上报  
2. 在 `_http_event_handler` 里把进度打印到串口  
3. 失败自动重试 3 次再停  
4. 切 HTTPS + 自家服务器做「真·远程 OTA」  

---

## 11. 下一步

- `03.development.usb_msc_ota` — U 盘 OTA  
- [MQTT](./P4C5例程详解-mqtt_tcp.md) — 联网消息  
- [HTTP Server](./P4C5例程详解-http_server.md) — 设备端 Web  

酷世DIY · Kevincoooool
