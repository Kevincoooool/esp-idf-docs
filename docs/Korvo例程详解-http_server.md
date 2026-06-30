# Korvo · HTTP Server

> **适用开发板**：Korvo S3 · **工程** `02.beginner.http_server` · **target** `esp32s3`  
> **前置**：[Wi-Fi Station](./Korvo例程详解-wifi_station.md)

板载 **HTTP 服务器**（默认 80 端口），演示 GET/POST 等 URI handler；浏览器同网段访问，屏幕显示 IP 与服务状态。与 P4C5 例程 **API 相同、target 不同**。

---

## 1. 这个例子在干嘛

1. Wi-Fi 获 IP  
2. `httpd_start()` + 注册多个 handler（hello/echo/ctrl 等，以 `main/main.c` 为准）  
3. 串口打印请求头、query、POST body  
4. 屏幕显示 `http://<IP>` 和 running 状态  

---

## 2. 编译烧录

```powershell
cd Korvo_Firmware\02.beginner.http_server
idf.py set-target esp32s3
idf.py menuconfig    # Wi-Fi
idf.py build flash monitor
```

---

## 3. 浏览器测试

假设屏幕显示 `http://192.168.1.50`：

| 路径 | 方法 | 预期 |
|------|------|------|
| `/hello` | GET | Hello World |
| `/hello?name=test` | GET | 串口见 query |
| `/echo` | POST | body 回显 |

具体路由以工程源码注册为准；P4C5 版文档更细，见 [P4C5 HTTP Server](./P4C5例程详解-http_server.md) 对照学习 handler 写法。

---

## 4. 关键文件

| 文件 | 作用 |
|------|------|
| `main/main.c` | 全部 httpd 逻辑 |

---

## 5. 重点 API

```text
httpd_start / httpd_stop
httpd_register_uri_handler / unregister
httpd_req_get_url_query_str
httpd_req_recv
httpd_resp_send / send_chunk
```

**URI handler** = 路径 + HTTP 方法 + 回调函数。

---

## 6. 常见问题

| 现象 | 处理 |
|------|------|
| 无法访问 | 同 Wi-Fi；IP 是否正确 |
| Korvo bin 烧到 P4 | **禁止**，target 不同 |

---

## 7. 动手改造

1. 新增 `/api/led` 控制 GPIO  
2. 返回 JSON 状态  
3. 学客户端：[http_request](./Korvo例程详解-http_server.md) 同仓库 `02.beginner.http_request`  

---

## 8. 下一步

[OTA](./Korvo例程详解-ota.md) · [camera_webserver](./Korvo例程详解-camera_webserver.md)

酷世DIY · Kevincoooool
