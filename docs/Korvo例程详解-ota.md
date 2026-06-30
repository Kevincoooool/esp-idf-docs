# Korvo · OTA 空中升级（HTTPS）

> **适用开发板**：ESP32-S3 Korvo 2 V3  
> **工程**：`Korvo_Firmware/02.beginner.ota`  
> **target**：`esp32s3`  
> **前置**：[Wi-Fi Station](./Korvo例程详解-wifi_station.md)  
> **方式**：**HTTPS** 下载固件，`esp_https_ota` + 证书校验  

与 P4C5 仓库的 **HTTP + HFS 本地测试** 流程不同；Korvo 走标准 **HTTPS OTA**，适合有 HTTPS 文件服务器或云 OTA 的场景。P4 见 [P4C5 OTA](./P4C5例程详解-ota.md)。

---

## 1. 这个例子在干嘛

1. 连 Wi-Fi（S3 **内置 Wi-Fi**，无协处理器等待）  
2. 从 menuconfig 配置的 **HTTPS URL** 下载新 app  
3. 写入备用 OTA 分区，校验后重启  
4. 串口打印 **当前固件 SHA-256**；屏幕显示连接/下载/失败/完成状态  

分区表常用 **`PARTITION_TABLE_TWO_OTA_LARGE`**：双 OTA 分区交替启动。

---

## 2. 编译烧录

```powershell
cd Korvo_Firmware\02.beginner.ota
idf.py set-target esp32s3
idf.py menuconfig
idf.py build flash monitor
```

日常用 **左侧 USB（CH9102）** 下载与 log。

---

## 3. menuconfig 必配

**Example Connection Configuration**

- WiFi SSID / Password  

**Example Configuration**

| 项 | 说明 |
|----|------|
| `EXAMPLE_FIRMWARE_UPGRADE_URL` | HTTPS bin 地址，如 `https://192.168.0.3:8070/simple_ota.bin` |
| `EXAMPLE_USE_CERT_BUNDLE` | 访问公网 HTTPS 时可开 **证书包** |
| `EXAMPLE_SKIP_COMMON_NAME_CHECK` | 自签证书内网测试时可开（生产慎用） |

**Component config → ESP HTTPS OTA**

- HTTPS 模式 **不要** 开 Allow HTTP（除非刻意测 HTTP）  

自签服务器需把 CA 放到 `server_certs/` 或配置 bundle。

---

## 4. 准备 OTA 服务器（简要）

**内网 HTTPS 示例思路**

1. PC 上用 nginx/caddy 托管 `build/simple_ota.bin`  
2. 配置 TLS 证书（自签或 Let's Encrypt）  
3. URL 与 menuconfig 一致  
4. 先浏览器下载验证，再让板子 OTA  

**验证升级**

1. 首次 `idf.py flash` 烧入版本 A  
2. 改版本号或代码后 **只更新服务器上的 bin**  
3. 板子 **RESET** 触发 OTA（不要每次 USB flash 覆盖测试）  
4. 重启后 SHA-256 变化即成功  

---

## 5. 关键文件

| 文件 | 作用 |
|------|------|
| `main/simple_ota_example.c` | OTA 任务、`esp_https_ota`、SHA 打印 |
| `main/Kconfig.projbuild` | URL、证书选项 |
| `server_certs/ca_cert.pem` | 自定义 CA |

Korvo 工程结构与 ESP-IDF 官方 `simple_ota_example` 相近，可对照 IDF 文档。

---

## 6. 核心流程

```text
app_main
  → nvs
  → get_sha256_of_partitions()    // 串口
  → example_connect()             // Wi-Fi
  └── simple_ota_example_task
        → esp_http_client_config { .url = CONFIG_EXAMPLE_FIRMWARE_UPGRADE_URL }
        → esp_https_ota()
        → esp_restart()
```

可选：`EXAMPLE_FIRMWARE_UPGRADE_BIND_IF` 绑定 STA 网卡；TLS 动态缓冲等高级项见 Kconfig。

---

## 7. 屏幕与串口

- 屏幕：连接、下载进度/状态、错误  
- 串口：bootloader/app hash、OTA 各阶段 log、`ESP_HTTPS_OTA` 错误码  

---

## 8. 常见问题

| 现象 | 处理 |
|------|------|
| TLS 失败 | 证书/URL；CN 检查；时间是否正确（SNTP） |
| 404 | 服务器路径与 bin 文件名 |
| 升级后无变化 | 服务器 bin 未更新；误用 flash 而非 RESET |
| 分区错误 | `erase-flash` 后重烧；确认 two_ota 分区表 |
| Wi-Fi 失败 | 同 [wifi_station](./Korvo例程详解-wifi_station.md) |

---

## 9. 动手改造

1. OTA 成功后 MQTT 上报新版本  
2. 失败重试 3 次  
3. 屏幕显示下载百分比（若未实现可参考 P4 OTA 的 HTTP 事件 handler）  

---

## 10. 下一步

[HTTP Server](./Korvo例程详解-http_server.md) · [MQTT](./Korvo例程详解-mqtt_tcp.md)

酷世DIY · Kevincoooool
