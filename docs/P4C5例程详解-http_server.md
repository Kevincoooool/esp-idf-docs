# P4C5 · HTTP Server

> **适用开发板**：P4C5 · **工程** `02.beginner.http_server` · **target** `esp32p4`  
> **前置**：[Wi-Fi Station](./P4C5例程详解-wifi_station.md)（等 C5 就绪）

在开发板上跑 **HTTP 服务器**（默认 80 端口），处理浏览器/手机的 GET、POST、PUT；屏幕显示访问 URL 和路由列表。是 Web 配网页、REST API、本地控制面板的基础。

---

## 1. 这个例子在干嘛

1. `example_connect()` 连 Wi-Fi 获 IP  
2. `httpd_start()` 创建服务器  
3. 注册多个 **URI handler**（路径 + HTTP 方法 + 回调）  
4. Wi-Fi 断开停服、重连自动重启  
5. LVGL 显示 `http://<IP>:80` 和已注册路径  

---

## 2. 编译烧录

```powershell
cd P4_C5_4.3_Firmware\02.beginner.http_server
idf.py set-target esp32p4
idf.py menuconfig
idf.py build flash monitor
```

### menuconfig

**Example Connection Configuration** → WiFi SSID / Password  

**Example Configuration（可选）**

| 项 | 说明 |
|----|------|
| Basic Authentication | HTTP 基础认证，默认关 |
| Enable SSE | Server-Sent Events 推送，默认关 |

---

## 3. 浏览器测试

屏幕第一行即访问地址，例如 `http://192.168.1.105:80`。手机/PC 须与板子 **同一 Wi-Fi**。

| URL | 方法 | 效果 |
|-----|------|------|
| `/hello` | GET | 返回 `Hello World!` |
| `/hello?query1=test` | GET | 串口打印 query 参数 |
| `/echo` | POST | body 原样 chunk 回显 |
| `/ctrl` | PUT | body=`0` 注销 hello/echo；非 0 恢复 |
| `/any` | ANY | 任意方法都返回 Hello |

---

## 4. 运行现象

**屏幕**

- Web 地址、URI 列表（`/hello /echo /ctrl`）、`web server running`  

**串口**

```text
Found header => Host: ...
Found URL query => query1=test
Received X bytes    // POST /echo
```

---

## 5. 关键文件

| 文件 | 作用 |
|------|------|
| `main/main.c` | 全部逻辑：Wi-Fi、httpd、各 handler |

---

## 6. 核心代码结构

```text
start_webserver()
  ├── httpd_config_t 默认 80 端口
  ├── httpd_start()
  └── httpd_register_uri_handler × N

hello_get_handler()   // GET：读 header、query，httpd_resp_send
echo_post_handler()   // POST：httpd_req_recv 分块，chunked 回应
ctrl_put_handler()    // PUT：动态 unregister/register handler
connect_handler()     // Wi-Fi 重连 → start_webserver
disconnect_handler()  // Wi-Fi 断 → stop_webserver
```

### httpd_uri_t

绑定 **uri 字符串**、**method 枚举**、**handler 函数指针**。同一 path 不同 method 需分别注册。

---

## 7. P4C5 注意

- Wi-Fi 经 C5，获 IP 可能比 S3 慢，等 `GOT IP` 再访问  
- **target 必须是 esp32p4**，勿烧 Korvo 的 bin  
- P4 仓库若无独立 `http_request` 入门，可对照 Korvo 仓库学 `esp_http_client`（客户端，与服务端互补）  

---

## 8. 常见问题

| 现象 | 处理 |
|------|------|
| 浏览器打不开 | IP 是否对；是否同网段；防火墙 |
| 404 | 路径大小写；handler 是否被 `/ctrl` 注销 |
| POST 无响应 | Content-Length；recv 缓冲区大小 |
| 断 Wi-Fi 后服务挂 | 看 disconnect_handler 是否执行；重连后 connect_handler |

---

## 9. 动手改造

1. 新增 `GET /status` 返回 JSON（heap、uptime）  
2. 开 Basic Auth 保护 `/ctrl`  
3. 把 LED/GPIO 控制绑到 POST `/api/led`  
4. 与 [MQTT](./P4C5例程详解-mqtt_tcp.md) 对比：HTTP 请求-响应 vs MQTT 长连接  

---

## 10. 下一步

[MQTT](./P4C5例程详解-mqtt_tcp.md) → [OTA](./P4C5例程详解-ota.md)

Korvo 同类例程见 [Korvo HTTP Server](./Korvo例程详解-http_server.md)。

酷世DIY · Kevincoooool
