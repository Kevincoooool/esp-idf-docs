


          
# ESP-IDF HTTP 客户端使用教程

## 目录
- [HTTP 简介](#http-简介)
- [基本使用](#基本使用)
- [文件上传](#文件上传)
- [文件下载](#文件下载)
- [高级用法](#高级用法)
- [常见问题](#常见问题)
- [性能优化](#性能优化)

## HTTP 简介

HTTP（超文本传输协议）是一种用于分布式、协作式和超媒体信息系统的应用层协议。ESP-IDF 提供了完整的 HTTP 客户端实现，支持 HTTP/1.1 协议，可以用于与 Web 服务器进行通信，执行 GET、POST 等请求，以及上传和下载文件。

ESP-IDF 的 HTTP 客户端基于 ESP-HTTP 库实现，提供了简单易用的 API，支持同步和异步操作，以及 TLS/SSL 安全连接。

## 基本使用

### 1. HTTP 客户端初始化

```c
#include <string.h>
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "esp_system.h"
#include "esp_wifi.h"
#include "esp_event.h"
#include "esp_log.h"
#include "nvs_flash.h"
#include "esp_http_client.h"

static const char *TAG = "HTTP_EXAMPLE";

// HTTP 事件处理函数
esp_err_t _http_event_handler(esp_http_client_event_t *evt)
{
    switch(evt->event_id) {
        case HTTP_EVENT_ERROR:
            ESP_LOGI(TAG, "HTTP_EVENT_ERROR");
            break;
        case HTTP_EVENT_ON_CONNECTED:
            ESP_LOGI(TAG, "HTTP_EVENT_ON_CONNECTED");
            break;
        case HTTP_EVENT_HEADER_SENT:
            ESP_LOGI(TAG, "HTTP_EVENT_HEADER_SENT");
            break;
        case HTTP_EVENT_ON_HEADER:
            ESP_LOGI(TAG, "HTTP_EVENT_ON_HEADER, key=%s, value=%s", evt->header_key, evt->header_value);
            break;
        case HTTP_EVENT_ON_DATA:
            ESP_LOGI(TAG, "HTTP_EVENT_ON_DATA, len=%d", evt->data_len);
            if (!esp_http_client_is_chunked_response(evt->client)) {
                // 打印响应数据
                printf("%.*s", evt->data_len, (char*)evt->data);
            }
            break;
        case HTTP_EVENT_ON_FINISH:
            ESP_LOGI(TAG, "HTTP_EVENT_ON_FINISH");
            break;
        case HTTP_EVENT_DISCONNECTED:
            ESP_LOGI(TAG, "HTTP_EVENT_DISCONNECTED");
            break;
    }
    return ESP_OK;
}

void http_get_example(void)
{
    // HTTP 客户端配置
    esp_http_client_config_t config = {
        .url = "http://example.com",  // 请求的 URL
        .event_handler = _http_event_handler,  // 事件处理函数
        .disable_auto_redirect = false,  // 自动重定向
        .max_redirection_count = 5,  // 最大重定向次数
        .timeout_ms = 5000,  // 超时时间（毫秒）
    };
    
    // 初始化 HTTP 客户端
    esp_http_client_handle_t client = esp_http_client_init(&config);
    
    // 执行 GET 请求
    esp_err_t err = esp_http_client_perform(client);
    
    if (err == ESP_OK) {
        // 获取 HTTP 状态码和内容长度
        int status_code = esp_http_client_get_status_code(client);
        int content_length = esp_http_client_get_content_length(client);
        ESP_LOGI(TAG, "HTTP GET 状态码 = %d, 内容长度 = %d", status_code, content_length);
    } else {
        ESP_LOGE(TAG, "HTTP GET 请求失败: %s", esp_err_to_name(err));
    }
    
    // 清理 HTTP 客户端
    esp_http_client_cleanup(client);
}
```

### 2. 发送 POST 请求

```c
void http_post_example(void)
{
    // HTTP 客户端配置
    esp_http_client_config_t config = {
        .url = "http://example.com/post",
        .event_handler = _http_event_handler,
        .method = HTTP_METHOD_POST,  // 设置 HTTP 方法为 POST
    };
    
    // 初始化 HTTP 客户端
    esp_http_client_handle_t client = esp_http_client_init(&config);
    
    // 设置 POST 数据
    const char *post_data = "field1=value1&field2=value2";
    esp_http_client_set_post_field(client, post_data, strlen(post_data));
    
    // 设置请求头
    esp_http_client_set_header(client, "Content-Type", "application/x-www-form-urlencoded");
    
    // 执行 POST 请求
    esp_err_t err = esp_http_client_perform(client);
    
    if (err == ESP_OK) {
        int status_code = esp_http_client_get_status_code(client);
        ESP_LOGI(TAG, "HTTP POST 状态码 = %d", status_code);
    } else {
        ESP_LOGE(TAG, "HTTP POST 请求失败: %s", esp_err_to_name(err));
    }
    
    // 清理 HTTP 客户端
    esp_http_client_cleanup(client);
}
```

### 3. 发送 JSON 数据

```c
void http_post_json_example(void)
{
    // HTTP 客户端配置
    esp_http_client_config_t config = {
        .url = "http://example.com/api",
        .event_handler = _http_event_handler,
        .method = HTTP_METHOD_POST,
    };
    
    // 初始化 HTTP 客户端
    esp_http_client_handle_t client = esp_http_client_init(&config);
    
    // 准备 JSON 数据
    const char *json_data = "{\"name\":\"ESP32\",\"value\":42}";
    
    // 设置 POST 数据
    esp_http_client_set_post_field(client, json_data, strlen(json_data));
    
    // 设置请求头
    esp_http_client_set_header(client, "Content-Type", "application/json");
    
    // 执行 POST 请求
    esp_err_t err = esp_http_client_perform(client);
    
    if (err == ESP_OK) {
        int status_code = esp_http_client_get_status_code(client);
        ESP_LOGI(TAG, "HTTP POST JSON 状态码 = %d", status_code);
    } else {
        ESP_LOGE(TAG, "HTTP POST JSON 请求失败: %s", esp_err_to_name(err));
    }
    
    // 清理 HTTP 客户端
    esp_http_client_cleanup(client);
}
```

## 文件上传

### 1. 上传文件（表单方式）

```c
void http_upload_file_example(void)
{
    // 打开要上传的文件
    FILE *file = fopen("/spiffs/example.txt", "r");
    if (file == NULL) {
        ESP_LOGE(TAG, "无法打开文件");
        return;
    }
    
    // 获取文件大小
    fseek(file, 0, SEEK_END);
    long file_size = ftell(file);
    fseek(file, 0, SEEK_SET);
    
    // 读取文件内容
    char *file_data = malloc(file_size + 1);
    if (file_data == NULL) {
        ESP_LOGE(TAG, "内存分配失败");
        fclose(file);
        return;
    }
    
    fread(file_data, 1, file_size, file);
    file_data[file_size] = '\0';
    fclose(file);
    
    // 构建多部分表单数据
    char *boundary = "----WebKitFormBoundaryXXXXXXXX";
    char *form_data_begin = "--" boundary "\r\nContent-Disposition: form-data; name=\"file\"; filename=\"example.txt\"\r\nContent-Type: text/plain\r\n\r\n";
    char *form_data_end = "\r\n--" boundary "--\r\n";
    
    // 计算总数据长度
    int total_len = strlen(form_data_begin) + file_size + strlen(form_data_end);
    char *post_data = malloc(total_len + 1);
    if (post_data == NULL) {
        ESP_LOGE(TAG, "内存分配失败");
        free(file_data);
        return;
    }
    
    // 组装表单数据
    sprintf(post_data, "%s%s%s", form_data_begin, file_data, form_data_end);
    free(file_data);
    
    // HTTP 客户端配置
    esp_http_client_config_t config = {
        .url = "http://example.com/upload",
        .event_handler = _http_event_handler,
        .method = HTTP_METHOD_POST,
    };
    
    // 初始化 HTTP 客户端
    esp_http_client_handle_t client = esp_http_client_init(&config);
    
    // 设置请求头
    char content_type[64];
    sprintf(content_type, "multipart/form-data; boundary=%s", boundary);
    esp_http_client_set_header(client, "Content-Type", content_type);
    
    // 设置 POST 数据
    esp_http_client_set_post_field(client, post_data, total_len);
    
    // 执行 POST 请求
    esp_err_t err = esp_http_client_perform(client);
    
    if (err == ESP_OK) {
        int status_code = esp_http_client_get_status_code(client);
        ESP_LOGI(TAG, "文件上传状态码 = %d", status_code);
    } else {
        ESP_LOGE(TAG, "文件上传失败: %s", esp_err_to_name(err));
    }
    
    // 清理资源
    esp_http_client_cleanup(client);
    free(post_data);
}
```

### 2. 分块上传大文件

```c
#define UPLOAD_BUFFER_SIZE 1024

void http_upload_large_file_example(const char *file_path, const char *upload_url)
{
    // 打开要上传的文件
    FILE *file = fopen(file_path, "r");
    if (file == NULL) {
        ESP_LOGE(TAG, "无法打开文件: %s", file_path);
        return;
    }
    
    // 获取文件大小
    fseek(file, 0, SEEK_END);
    long file_size = ftell(file);
    fseek(file, 0, SEEK_SET);
    
    ESP_LOGI(TAG, "开始上传文件，大小: %ld 字节", file_size);
    
    // HTTP 客户端配置
    esp_http_client_config_t config = {
        .url = upload_url,
        .event_handler = _http_event_handler,
        .method = HTTP_METHOD_POST,
        .buffer_size = UPLOAD_BUFFER_SIZE,
    };
    
    // 初始化 HTTP 客户端
    esp_http_client_handle_t client = esp_http_client_init(&config);
    
    // 设置请求头
    esp_http_client_set_header(client, "Content-Type", "application/octet-stream");
    
    // 打开连接但不发送请求
    esp_err_t err = esp_http_client_open(client, file_size);
    if (err != ESP_OK) {
        ESP_LOGE(TAG, "无法建立 HTTP 连接: %s", esp_err_to_name(err));
        fclose(file);
        esp_http_client_cleanup(client);
        return;
    }
    
    // 分块读取文件并上传
    char *buffer = malloc(UPLOAD_BUFFER_SIZE);
    if (buffer == NULL) {
        ESP_LOGE(TAG, "内存分配失败");
        fclose(file);
        esp_http_client_cleanup(client);
        return;
    }
    
    int read_len;
    int total_read = 0;
    
    while ((read_len = fread(buffer, 1, UPLOAD_BUFFER_SIZE, file)) > 0) {
        // 写入数据
        int write_len = esp_http_client_write(client, buffer, read_len);
        if (write_len < 0) {
            ESP_LOGE(TAG, "上传数据时出错");
            break;
        }
        
        total_read += read_len;
        ESP_LOGI(TAG, "已上传: %d / %ld 字节", total_read, file_size);
    }
    
    // 关闭文件
    fclose(file);
    free(buffer);
    
    // 获取响应
    int content_length = esp_http_client_fetch_headers(client);
    if (content_length < 0) {
        ESP_LOGE(TAG, "HTTP 客户端获取响应头失败");
    } else {
        int status_code = esp_http_client_get_status_code(client);
        ESP_LOGI(TAG, "文件上传完成，状态码: %d", status_code);
        
        // 读取响应数据
        if (content_length > 0) {
            char *response_buffer = malloc(content_length + 1);
            if (response_buffer != NULL) {
                int data_read = esp_http_client_read_response(client, response_buffer, content_length);
                if (data_read >= 0) {
                    response_buffer[data_read] = '\0';
                    ESP_LOGI(TAG, "服务器响应: %s", response_buffer);
                }
                free(response_buffer);
            }
        }
    }
    
    // 清理 HTTP 客户端
    esp_http_client_cleanup(client);
}
```

## 文件下载

### 1. 下载文件到内存

```c
void http_download_to_memory_example(const char *url)
{
    // HTTP 客户端配置
    esp_http_client_config_t config = {
        .url = url,
        .event_handler = _http_event_handler,
    };
    
    // 初始化 HTTP 客户端
    esp_http_client_handle_t client = esp_http_client_init(&config);
    
    // 执行 GET 请求
    esp_err_t err = esp_http_client_perform(client);
    
    if (err == ESP_OK) {
        int status_code = esp_http_client_get_status_code(client);
        int content_length = esp_http_client_get_content_length(client);
        
        ESP_LOGI(TAG, "HTTP GET 状态码 = %d, 内容长度 = %d", status_code, content_length);
        
        if (status_code == 200 && content_length > 0) {
            // 分配内存存储下载的数据
            char *download_data = malloc(content_length + 1);
            if (download_data == NULL) {
                ESP_LOGE(TAG, "内存分配失败");
                esp_http_client_cleanup(client);
                return;
            }
            
            // 重新执行请求以获取数据
            esp_http_client_set_method(client, HTTP_METHOD_GET);
            esp_http_client_open(client, 0);
            
            int data_read = esp_http_client_read_response(client, download_data, content_length);
            if (data_read >= 0) {
                download_data[data_read] = '\0';
                ESP_LOGI(TAG, "下载完成，数据长度: %d", data_read);
                
                // 这里可以处理下载的数据
                // ...
                
                // 示例：打印前 100 个字符
                ESP_LOGI(TAG, "数据预览: %.100s%s", download_data, data_read > 100 ? "..." : "");
            }
            
            free(download_data);
        }
    } else {
        ESP_LOGE(TAG, "HTTP GET 请求失败: %s", esp_err_to_name(err));
    }
    
    // 清理 HTTP 客户端
    esp_http_client_cleanup(client);
}
```

### 2. 下载文件到 SPIFFS

```c
#define MAX_HTTP_RECV_BUFFER 512
#define MAX_HTTP_OUTPUT_BUFFER 2048

void http_download_to_file_example(const char *url, const char *file_path)
{
    // HTTP 客户端配置
    esp_http_client_config_t config = {
        .url = url,
        .event_handler = _http_event_handler,
        .buffer_size = MAX_HTTP_RECV_BUFFER,
    };
    
    // 初始化 HTTP 客户端
    esp_http_client_handle_t client = esp_http_client_init(&config);
    
    // 打开文件准备写入
    FILE *file = fopen(file_path, "w");
    if (file == NULL) {
        ESP_LOGE(TAG, "无法创建文件: %s", file_path);
        esp_http_client_cleanup(client);
        return;
    }
    
    // 执行 GET 请求
    esp_err_t err = esp_http_client_perform(client);
    
    if (err == ESP_OK) {
        int status_code = esp_http_client_get_status_code(client);
        int content_length = esp_http_client_get_content_length(client);
        
        ESP_LOGI(TAG, "HTTP GET 状态码 = %d, 内容长度 = %d", status_code, content_length);
        
        if (status_code == 200) {
            // 重新执行请求以获取数据
            esp_http_client_set_method(client, HTTP_METHOD_GET);
            esp_http_client_open(client, 0);
            
            char *buffer = malloc(MAX_HTTP_OUTPUT_BUFFER);
            if (buffer == NULL) {
                ESP_LOGE(TAG, "内存分配失败");
                fclose(file);
                esp_http_client_cleanup(client);
                return;
            }
            
            // 读取数据并写入文件
            int read_len;
            int total_read = 0;
            
            while (1) {
                read_len = esp_http_client_read(client, buffer, MAX_HTTP_OUTPUT_BUFFER);
                if (read_len <= 0) {
                    break;
                }
                
                fwrite(buffer, 1, read_len, file);
                total_read += read_len;
                
                ESP_LOGI(TAG, "已下载: %d 字节", total_read);
            }
            
            ESP_LOGI(TAG, "文件下载完成: %s, 总大小: %d 字节", file_path, total_read);
            free(buffer);
        }
    } else {
        ESP_LOGE(TAG, "HTTP GET 请求失败: %s", esp_err_to_name(err));
    }
    
    // 关闭文件
    fclose(file);
    
    // 清理 HTTP 客户端
    esp_http_client_cleanup(client);
}
```

### 3. 断点续传下载

```c
void http_resume_download_example(const char *url, const char *file_path, int start_pos)
{
    // 打开文件准备追加
    FILE *file = fopen(file_path, start_pos > 0 ? "a" : "w");
    if (file == NULL) {
        ESP_LOGE(TAG, "无法打开文件: %s", file_path);
        return;
    }
    
    // HTTP 客户端配置
    esp_http_client_config_t config = {
        .url = url,
        .event_handler = _http_event_handler,
    };
    
    // 初始化 HTTP 客户端
    esp_http_client_handle_t client = esp_http_client_init(&config);
    
    // 设置断点续传的 Range 头
    if (start_pos > 0) {
        char range_header[32];
        sprintf(range_header, "bytes=%d-", start_pos);
        esp_http_client_set_header(client, "Range", range_header);
        ESP_LOGI(TAG, "断点续传，起始位置: %d", start_pos);
    }
    
    // 执行 GET 请求
    esp_err_t err = esp_http_client_perform(client);
    
    if (err == ESP_OK) {
        int status_code = esp_http_client_get_status_code(client);
        int content_length = esp_http_client_get_content_length(client);
        
        ESP_LOGI(TAG, "HTTP GET 状态码 = %d, 内容长度 = %d", status_code, content_length);
        
        if ((start_pos > 0 && status_code == 206) || (start_pos == 0 && status_code == 200)) {
            // 重新执行请求以获取数据
            esp_http_client_set_method(client, HTTP_METHOD_GET);
            
            if (start_pos > 0) {
                char range_header[32];
                sprintf(range_header, "bytes=%d-", start_pos);
                esp_http_client_set_header(client, "Range", range_header);
            }
            
            esp_http_client_open(client, 0);
            
            char *buffer = malloc(MAX_HTTP_OUTPUT_BUFFER);
            if (buffer == NULL) {
                ESP_LOGE(TAG, "内存分配失败");
                fclose(file);
                esp_http_client_cleanup(client);
                return;
            }
            
            // 读取数据并写入文件
            int read_len;
            int total_read = 0;
            
            while (1) {
                read_len = esp_http_client_read(client, buffer, MAX_HTTP_OUTPUT_BUFFER);
                if (read_len <= 0) {
                    break;
                }
                
                fwrite(buffer, 1, read_len, file);
                total_read += read_len;
                
                ESP_LOGI(TAG, "已下载: %d 字节", total_read);
            }
            
            ESP_LOGI(TAG, "文件下载完成: %s, 本次下载: %d 字节, 总大小: %d 字节", 
                    file_path, total_read, start_pos + total_read);
            free(buffer);
        } else {
            ESP_LOGE(TAG, "服务器不支持断点续传或请求失败");
        }
    } else {
        ESP_LOGE(TAG, "HTTP GET 请求失败: %s", esp_err_to_name(err));
    }
    
    // 关闭文件
    fclose(file);
    
    // 清理 HTTP 客户端
    esp_http_client_cleanup(client);
}
```

## 高级用法

### 1. 使用 HTTPS 安全连接

```c
void https_request_example(void)
{
    // HTTPS 客户端配置
    esp_http_client_config_t config = {
        .url = "https://www.howsmyssl.com/a/check",  // HTTPS URL
        .event_handler = _http_event_handler,
        .cert_pem = howsmyssl_com_root_cert_pem_start,  // 服务器证书
        .transport_type = HTTP_TRANSPORT_OVER_SSL,  // 使用 SSL
    };
    
    // 初始化 HTTP 客户端
    esp_http_client_handle_t client = esp_http_client_init(&config);
    
    // 执行 HTTPS 请求
    esp_err_t err = esp_http_client_perform(client);
    
    if (err == ESP_OK) {
        int status_code = esp_http_client_get_status_code(client);
        ESP_LOGI(TAG, "HTTPS 请求状态码 = %d", status_code);
    } else {
        ESP_LOGE(TAG, "HTTPS 请求失败: %s", esp_err_to_name(err));
    }
    
    // 清理 HTTP 客户端
    esp_http_client_cleanup(client);
}
```

### 2. 异步 HTTP 请求

```c
// 异步 HTTP 任务
void http_async_task(void *pvParameters)
{
    esp_http_client_handle_t client = (esp_http_client_handle_t)pvParameters;
    
    // 执行 HTTP 请求
    esp_err_t err = esp_http_client_perform(client);
    
    if (err == ESP_OK) {
        int status_code = esp_http_client_get_status_code(client);
        ESP_LOGI(TAG, "异步 HTTP 请求状态码 = %d", status_code);
    } else {
        ESP_LOGE(TAG, "异步 HTTP 请求失败: %s", esp_err_to_name(err));
    }
    
    // 清理 HTTP 客户端
    esp_http_client_cleanup(client);
    
    // 删除任务
    vTaskDelete(NULL);
}

void http_async_request_example(const char *url)
{
    // HTTP 客户端配置
    esp_http_client_config_t config = {
        .url = url,
        .event_handler = _http_event_handler,
    };
    
    // 初始化 HTTP 客户端
    esp_http_client_handle_t client = esp_http_client_init(&config);
    
    // 创建异步任务
    xTaskCreate(http_async_task, "http_async_task", 4096, (void*)client, 5, NULL);
    
    ESP_LOGI(TAG, "异步 HTTP 请求已启动");
}
```

### 3. 使用 HTTP 基本认证

```c
void http_basic_auth_example(void)
{
    // HTTP 客户端配置
    esp_http_client_config_t config = {
        .url = "http://example.com/protected",
        .event_handler = _http_event_handler,
        .username = "user",  // 用户名
        .password = "pass",  // 密码
        .auth_type = HTTP_AUTH_TYPE_BASIC,  // 基本认证
    };
    
    // 初始化 HTTP 客户端
    esp_http_client_handle_t client = esp_http_client_init(&config);
    
    // 执行 GET 请求
    esp_err_t err = esp_http_client_perform(client);
    
    if (err == ESP_OK) {
        int status_code = esp_http_client_get_status_code(client);
        ESP_LOGI(TAG, "HTTP 基本认证请求状态码 = %d", status_code);
    } else {
        ESP_LOGE(TAG, "HTTP 基本认证请求失败: %s", esp_err_to_name(err));
    }
    
    // 清理 HTTP 客户端
    esp_http_client_cleanup(client);
}
```

### 4. 使用 HTTP 代理

```c
void http_proxy_example(void)
{
    // HTTP 客户端配置
    esp_http_client_config_t config = {
        .url = "http://example.com",
        .event_handler = _http_event_handler,
        .proxy_host = "proxy.example.com",  // 代理服务器主机
        .proxy_port = 3128,  // 代理服务器端口
    };
    
    // 初始化 HTTP 客户端
    esp_http_client_handle_t client = esp_http_client_init(&config);
    
    // 执行 GET 请求
    esp_err_t err = esp_http_client_perform(client);
    
    if (err == ESP_OK) {
        int status_code = esp_http_client_get_status_code(client);
        ESP_LOGI(TAG, "HTTP 代理请求状态码 = %d", status_code);
    } else {
        ESP_LOGE(TAG, "HTTP 代理请求失败: %s", esp_err_to_name(err));
    }
    
    // 清理 HTTP 客户端
    esp_http_client_cleanup(client);
}
```



          
### 3. 使用 HTTP 基本认证

```c
void http_basic_auth_example(void)
{
    // HTTP 客户端配置
    esp_http_client_config_t config = {
        .url = "http://example.com/protected",
        .event_handler = _http_event_handler,
        .username = "user",  // 用户名
        .password = "pass",  // 密码
        .auth_type = HTTP_AUTH_TYPE_BASIC,  // 基本认证
    };
    
    // 初始化 HTTP 客户端
    esp_http_client_handle_t client = esp_http_client_init(&config);
    
    // 执行请求
    esp_err_t err = esp_http_client_perform(client);
    
    if (err == ESP_OK) {
        int status_code = esp_http_client_get_status_code(client);
        ESP_LOGI(TAG, "HTTP 基本认证请求状态码 = %d", status_code);
    } else {
        ESP_LOGE(TAG, "HTTP 基本认证请求失败: %s", esp_err_to_name(err));
    }
    
    // 清理 HTTP 客户端
    esp_http_client_cleanup(client);
}
```

## 常见问题

### 1. 连接问题

如果 HTTP 连接失败，可能的原因和解决方法：

- **网络问题**：
  - 确保 WiFi 已连接并能访问互联网
  - 检查服务器地址和端口是否正确
  - 确认防火墙未阻止 HTTP 流量

- **DNS 解析问题**：
  - 检查 DNS 服务器配置
  - 尝试使用 IP 地址代替域名

- **SSL/TLS 问题**（HTTPS）：
  - 确保证书配置正确
  - 检查证书是否过期
  - 验证证书链是否完整

### 2. 内存问题

如果遇到内存相关问题：

- **内存泄漏**：
  - 确保每次调用 `esp_http_client_init` 后都有对应的 `esp_http_client_cleanup`
  - 正确释放动态分配的缓冲区
  - 使用适当的缓冲区大小

- **堆内存不足**：
  - 减小缓冲区大小
  - 分块处理大文件
  - 增加堆内存配置

### 3. 超时问题

处理超时相关问题：

```c
void http_timeout_handling_example(void)
{
    // HTTP 客户端配置
    esp_http_client_config_t config = {
        .url = "http://example.com",
        .event_handler = _http_event_handler,
        .timeout_ms = 10000,  // 设置超时时间为 10 秒
        .buffer_size = 2048,  // 设置缓冲区大小
    };
    
    // 初始化 HTTP 客户端
    esp_http_client_handle_t client = esp_http_client_init(&config);
    
    // 执行请求
    esp_err_t err = esp_http_client_perform(client);
    
    if (err == ESP_ERR_HTTP_CONNECT) {
        ESP_LOGE(TAG, "连接超时");
    } else if (err == ESP_ERR_HTTP_TIMEOUT) {
        ESP_LOGE(TAG, "请求超时");
    }
    
    // 清理 HTTP 客户端
    esp_http_client_cleanup(client);
}
```

## 性能优化

### 1. 连接复用

```c
void http_connection_reuse_example(void)
{
    // HTTP 客户端配置
    esp_http_client_config_t config = {
        .url = "http://example.com",
        .event_handler = _http_event_handler,
        .keep_alive_enable = true,  // 启用 keep-alive
    };
    
    // 初始化 HTTP 客户端
    esp_http_client_handle_t client = esp_http_client_init(&config);
    
    // 多次使用同一连接
    for (int i = 0; i < 5; i++) {
        // 设置不同的 URL 路径
        char url[64];
        sprintf(url, "http://example.com/api/%d", i);
        esp_http_client_set_url(client, url);
        
        // 执行请求
        esp_err_t err = esp_http_client_perform(client);
        
        if (err == ESP_OK) {
            int status_code = esp_http_client_get_status_code(client);
            ESP_LOGI(TAG, "请求 %d 状态码 = %d", i, status_code);
        }
        
        // 等待一段时间再发送下一个请求
        vTaskDelay(pdMS_TO_TICKS(1000));
    }
    
    // 清理 HTTP 客户端
    esp_http_client_cleanup(client);
}
```

### 2. 压缩传输

```c
void http_compression_example(void)
{
    // HTTP 客户端配置
    esp_http_client_config_t config = {
        .url = "http://example.com",
        .event_handler = _http_event_handler,
        .buffer_size = 2048,
    };
    
    // 初始化 HTTP 客户端
    esp_http_client_handle_t client = esp_http_client_init(&config);
    
    // 添加支持压缩的请求头
    esp_http_client_set_header(client, "Accept-Encoding", "gzip, deflate");
    
    // 执行请求
    esp_err_t err = esp_http_client_perform(client);
    
    if (err == ESP_OK) {
        // 检查响应是否被压缩
        char *content_encoding = NULL;
        esp_http_client_get_header(client, "Content-Encoding", &content_encoding);
        
        if (content_encoding != NULL) {
            ESP_LOGI(TAG, "响应使用 %s 压缩", content_encoding);
        }
    }
    
    // 清理 HTTP 客户端
    esp_http_client_cleanup(client);
}
```

### 3. 批量请求处理

```c
void http_batch_request_example(void)
{
    // 准备多个请求
    const char *urls[] = {
        "http://example.com/api/1",
        "http://example.com/api/2",
        "http://example.com/api/3"
    };
    int num_requests = sizeof(urls) / sizeof(urls[0]);
    
    // HTTP 客户端配置
    esp_http_client_config_t config = {
        .event_handler = _http_event_handler,
        .keep_alive_enable = true,  // 启用 keep-alive
    };
    
    // 初始化 HTTP 客户端
    esp_http_client_handle_t client = esp_http_client_init(&config);
    
    // 批量处理请求
    for (int i = 0; i < num_requests; i++) {
        // 设置 URL
        esp_http_client_set_url(client, urls[i]);
        
        // 执行请求
        esp_err_t err = esp_http_client_perform(client);
        
        if (err == ESP_OK) {
            int status_code = esp_http_client_get_status_code(client);
            ESP_LOGI(TAG, "请求 %d 状态码 = %d", i, status_code);
        } else {
            ESP_LOGE(TAG, "请求 %d 失败: %s", i, esp_err_to_name(err));
        }
    }
    
    // 清理 HTTP 客户端
    esp_http_client_cleanup(client);
}
```

## 总结

本教程详细介绍了如何在 ESP-IDF 中使用 HTTP 客户端，包括：

1. **基本使用**：
   - GET 和 POST 请求
   - JSON 数据处理
   - 文件上传和下载

2. **高级功能**：
   - HTTPS 安全连接
   - 异步请求
   - 基本认证
   - 断点续传

3. **性能优化**：
   - 连接复用
   - 压缩传输
   - 批量请求处理

4. **常见问题处理**：
   - 连接问题
   - 内存管理
   - 超时处理

通过合理使用这些功能，可以构建稳定可靠的 HTTP 通信应用。记住要正确处理错误情况，合理管理内存，并在必要时使用性能优化技术。

## 参考资料

- [ESP-IDF HTTP 客户端文档](https://docs.espressif.com/projects/esp-idf/zh_CN/latest/esp32/api-reference/protocols/esp_http_client.html)
- [ESP-IDF 编程指南](https://docs.espressif.com/projects/esp-idf/zh_CN/latest/esp32/get-started/index.html)
- [ESP32 技术参考手册](https://www.espressif.com/sites/default/files/documentation/esp32_technical_reference_manual_cn.pdf)

        