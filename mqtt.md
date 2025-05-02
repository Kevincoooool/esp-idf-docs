


          
# ESP-IDF MQTT 客户端使用教程

## 目录
- [MQTT 简介](#mqtt-简介)
- [基本使用](#基本使用)
- [MQTT 重连机制](#mqtt-重连机制)
- [发布和订阅消息](#发布和订阅消息)
- [高级用法](#高级用法)
- [常见问题](#常见问题)
- [性能优化](#性能优化)

## MQTT 简介

MQTT（Message Queuing Telemetry Transport，消息队列遥测传输）是一种轻量级的发布/订阅消息传输协议，专为受限设备和低带宽、高延迟或不可靠的网络设计。ESP-IDF 提供了基于 ESP-MQTT 库的完整 MQTT 客户端实现，支持 MQTT 3.1 和 3.1.1 版本，并提供 QoS 0、1、2 服务质量级别。

### MQTT 的主要特点：
- 轻量级：协议简单，占用资源少
- 发布/订阅模式：解耦消息发送者和接收者
- 支持 QoS（服务质量）：确保消息可靠传递
- 保留消息：新订阅者可以接收之前的消息
- 遗嘱消息：客户端异常断开时自动发送通知

## 基本使用

### 1. MQTT 客户端初始化

```c
#include <stdio.h>
#include <stdint.h>
#include <stddef.h>
#include <string.h>
#include "esp_wifi.h"
#include "esp_system.h"
#include "nvs_flash.h"
#include "esp_event.h"
#include "esp_netif.h"

#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "freertos/semphr.h"
#include "freertos/queue.h"

#include "lwip/sockets.h"
#include "lwip/dns.h"
#include "lwip/netdb.h"

#include "esp_log.h"
#include "mqtt_client.h"

static const char *TAG = "MQTT_EXAMPLE";

static esp_mqtt_client_handle_t mqtt_client = NULL;

static void mqtt_event_handler(void *handler_args, esp_event_base_t base, int32_t event_id, void *event_data)
{
    esp_mqtt_event_handle_t event = event_data;
    esp_mqtt_client_handle_t client = event->client;
    
    // 处理 MQTT 事件
    switch (event->event_id) {
        case MQTT_EVENT_CONNECTED:
            ESP_LOGI(TAG, "MQTT_EVENT_CONNECTED");
            // 连接成功后订阅主题
            esp_mqtt_client_subscribe(client, "my_topic", 0);
            break;
            
        case MQTT_EVENT_DISCONNECTED:
            ESP_LOGI(TAG, "MQTT_EVENT_DISCONNECTED");
            break;
            
        case MQTT_EVENT_SUBSCRIBED:
            ESP_LOGI(TAG, "MQTT_EVENT_SUBSCRIBED, msg_id=%d", event->msg_id);
            break;
            
        case MQTT_EVENT_UNSUBSCRIBED:
            ESP_LOGI(TAG, "MQTT_EVENT_UNSUBSCRIBED, msg_id=%d", event->msg_id);
            break;
            
        case MQTT_EVENT_PUBLISHED:
            ESP_LOGI(TAG, "MQTT_EVENT_PUBLISHED, msg_id=%d", event->msg_id);
            break;
            
        case MQTT_EVENT_DATA:
            ESP_LOGI(TAG, "MQTT_EVENT_DATA");
            printf("TOPIC=%.*s\r\n", event->topic_len, event->topic);
            printf("DATA=%.*s\r\n", event->data_len, event->data);
            break;
            
        case MQTT_EVENT_ERROR:
            ESP_LOGI(TAG, "MQTT_EVENT_ERROR");
            if (event->error_handle->error_type == MQTT_ERROR_TYPE_TCP_TRANSPORT) {
                ESP_LOGI(TAG, "Last error code reported from esp-tls: 0x%x", event->error_handle->esp_tls_last_esp_err);
                ESP_LOGI(TAG, "Last tls stack error number: 0x%x", event->error_handle->esp_tls_stack_err);
                ESP_LOGI(TAG, "Last captured errno : %d (%s)",  event->error_handle->esp_transport_sock_errno,
                       strerror(event->error_handle->esp_transport_sock_errno));
            } else if (event->error_handle->error_type == MQTT_ERROR_TYPE_CONNECTION_REFUSED) {
                ESP_LOGI(TAG, "Connection refused error: 0x%x", event->error_handle->connect_return_code);
            } else {
                ESP_LOGW(TAG, "Unknown error type: 0x%x", event->error_handle->error_type);
            }
            break;
            
        default:
            ESP_LOGI(TAG, "Other event id:%d", event->event_id);
            break;
    }
}

void mqtt_app_start(void)
{
    // MQTT 客户端配置
    esp_mqtt_client_config_t mqtt_cfg = {
        .broker.address.uri = "mqtt://mqtt.eclipseprojects.io",  // MQTT 服务器地址
        .broker.address.port = 1883,                             // MQTT 服务器端口
        .credentials.username = NULL,                            // 用户名（如果需要）
        .credentials.authentication.password = NULL,             // 密码（如果需要）
        .session.keepalive = 120,                                // 保活时间（秒）
        .session.disable_clean_session = false,                  // 是否禁用清理会话
        .network.reconnect_timeout_ms = 10000,                   // 重连超时时间（毫秒）
    };
    
    // 创建 MQTT 客户端
    mqtt_client = esp_mqtt_client_init(&mqtt_cfg);
    
    // 注册 MQTT 事件处理函数
    esp_mqtt_client_register_event(mqtt_client, ESP_EVENT_ANY_ID, mqtt_event_handler, NULL);
    
    // 启动 MQTT 客户端
    esp_mqtt_client_start(mqtt_client);
}

void app_main(void)
{
    // 初始化 NVS
    esp_err_t ret = nvs_flash_init();
    if (ret == ESP_ERR_NVS_NO_FREE_PAGES || ret == ESP_ERR_NVS_NEW_VERSION_FOUND) {
        ESP_ERROR_CHECK(nvs_flash_erase());
        ret = nvs_flash_init();
    }
    ESP_ERROR_CHECK(ret);
    
    // 初始化 WiFi（假设已经实现）
    wifi_init_sta();
    
    // 启动 MQTT 客户端
    mqtt_app_start();
}
```

## MQTT 重连机制

### 1. 基本重连机制

ESP-IDF 的 MQTT 客户端内置了自动重连机制，但我们可以增强它以处理更复杂的场景：

```c
#define MQTT_RECONNECT_DELAY_MS  5000  // 重连延迟（毫秒）
#define MQTT_MAX_RECONNECT_COUNT 10    // 最大重连次数

static int mqtt_reconnect_count = 0;
static bool mqtt_connected = false;
static TimerHandle_t mqtt_reconnect_timer = NULL;

// MQTT 重连定时器回调函数
static void mqtt_reconnect_timer_callback(TimerHandle_t xTimer)
{
    if (!mqtt_connected && mqtt_reconnect_count < MQTT_MAX_RECONNECT_COUNT) {
        ESP_LOGI(TAG, "尝试重新连接 MQTT 服务器...");
        esp_mqtt_client_reconnect(mqtt_client);
        mqtt_reconnect_count++;
    } else if (mqtt_reconnect_count >= MQTT_MAX_RECONNECT_COUNT) {
        ESP_LOGW(TAG, "达到最大重连次数，重启设备...");
        esp_restart();
    }
}

// 增强版 MQTT 事件处理函数
static void mqtt_event_handler_enhanced(void *handler_args, esp_event_base_t base, int32_t event_id, void *event_data)
{
    esp_mqtt_event_handle_t event = event_data;
    esp_mqtt_client_handle_t client = event->client;
    
    switch (event->event_id) {
        case MQTT_EVENT_CONNECTED:
            ESP_LOGI(TAG, "MQTT_EVENT_CONNECTED");
            mqtt_connected = true;
            mqtt_reconnect_count = 0;  // 重置重连计数
            
            // 停止重连定时器
            if (mqtt_reconnect_timer != NULL) {
                xTimerStop(mqtt_reconnect_timer, 0);
            }
            
            // 连接成功后订阅主题
            esp_mqtt_client_subscribe(client, "my_topic", 1);  // QoS 1
            break;
            
        case MQTT_EVENT_DISCONNECTED:
            ESP_LOGI(TAG, "MQTT_EVENT_DISCONNECTED");
            mqtt_connected = false;
            
            // 启动重连定时器
            if (mqtt_reconnect_timer != NULL) {
                xTimerStart(mqtt_reconnect_timer, 0);
            }
            break;
            
        // ... 其他事件处理与之前相同 ...
    }
}

// 初始化 MQTT 重连机制
void init_mqtt_reconnect(void)
{
    // 创建重连定时器
    mqtt_reconnect_timer = xTimerCreate(
        "mqtt_reconnect_timer",
        pdMS_TO_TICKS(MQTT_RECONNECT_DELAY_MS),
        pdTRUE,  // 自动重载
        NULL,
        mqtt_reconnect_timer_callback);
}

// 修改后的 MQTT 启动函数
void mqtt_app_start_enhanced(void)
{
    // MQTT 客户端配置
    esp_mqtt_client_config_t mqtt_cfg = {
        .broker.address.uri = "mqtt://mqtt.eclipseprojects.io",
        .broker.address.port = 1883,
        .credentials.username = NULL,
        .credentials.authentication.password = NULL,
        .session.keepalive = 120,
        .session.disable_clean_session = false,
        .network.reconnect_timeout_ms = 10000,
        .network.transport = MQTT_TRANSPORT_OVER_TCP,  // 使用 TCP 传输
    };
    
    // 创建 MQTT 客户端
    mqtt_client = esp_mqtt_client_init(&mqtt_cfg);
    
    // 初始化重连机制
    init_mqtt_reconnect();
    
    // 注册 MQTT 事件处理函数
    esp_mqtt_client_register_event(mqtt_client, ESP_EVENT_ANY_ID, mqtt_event_handler_enhanced, NULL);
    
    // 启动 MQTT 客户端
    esp_mqtt_client_start(mqtt_client);
}
```

### 2. 高级重连机制（指数退避）

```c
#define MQTT_RECONNECT_INTERVAL_MIN  1000   // 最小重连间隔（毫秒）
#define MQTT_RECONNECT_INTERVAL_MAX  60000  // 最大重连间隔（毫秒）
#define MQTT_RECONNECT_FACTOR        2      // 退避因子

static uint32_t mqtt_current_reconnect_interval = MQTT_RECONNECT_INTERVAL_MIN;

// MQTT 重连定时器回调函数（指数退避）
static void mqtt_reconnect_timer_callback_backoff(TimerHandle_t xTimer)
{
    if (!mqtt_connected) {
        ESP_LOGI(TAG, "尝试重新连接 MQTT 服务器...");
        esp_mqtt_client_reconnect(mqtt_client);
        
        // 更新下一次重连间隔（指数退避）
        mqtt_current_reconnect_interval *= MQTT_RECONNECT_FACTOR;
        if (mqtt_current_reconnect_interval > MQTT_RECONNECT_INTERVAL_MAX) {
            mqtt_current_reconnect_interval = MQTT_RECONNECT_INTERVAL_MAX;
        }
        
        // 更新定时器周期
        xTimerChangePeriod(mqtt_reconnect_timer, 
                          pdMS_TO_TICKS(mqtt_current_reconnect_interval), 0);
    }
}

// 在连接成功时重置重连间隔
static void reset_mqtt_reconnect_interval(void)
{
    mqtt_current_reconnect_interval = MQTT_RECONNECT_INTERVAL_MIN;
}
```

## 发布和订阅消息

### 1. 发布消息

```c
// 发布消息到指定主题
void mqtt_publish_message(const char *topic, const char *data, int qos)
{
    if (mqtt_client == NULL || !mqtt_connected) {
        ESP_LOGW(TAG, "MQTT 客户端未连接，无法发布消息");
        return;
    }
    
    int msg_id = esp_mqtt_client_publish(mqtt_client, topic, data, 0, qos, 0);
    ESP_LOGI(TAG, "发布消息成功，msg_id=%d", msg_id);
}

// 发布保留消息
void mqtt_publish_retained_message(const char *topic, const char *data, int qos)
{
    if (mqtt_client == NULL || !mqtt_connected) {
        ESP_LOGW(TAG, "MQTT 客户端未连接，无法发布消息");
        return;
    }
    
    int msg_id = esp_mqtt_client_publish(mqtt_client, topic, data, 0, qos, 1);  // 最后一个参数为 1 表示保留消息
    ESP_LOGI(TAG, "发布保留消息成功，msg_id=%d", msg_id);
}

// 示例：发布传感器数据
void publish_sensor_data(float temperature, float humidity)
{
    char data[100];
    snprintf(data, sizeof(data), "{\"temperature\":%.1f,\"humidity\":%.1f}", temperature, humidity);
    
    mqtt_publish_message("sensors/data", data, 1);  // 使用 QoS 1
}
```

### 2. 订阅主题

```c
// 订阅主题
void mqtt_subscribe_topic(const char *topic, int qos)
{
    if (mqtt_client == NULL || !mqtt_connected) {
        ESP_LOGW(TAG, "MQTT 客户端未连接，无法订阅主题");
        return;
    }
    
    int msg_id = esp_mqtt_client_subscribe(mqtt_client, topic, qos);
    ESP_LOGI(TAG, "订阅主题 %s 成功，msg_id=%d", topic, msg_id);
}

// 取消订阅主题
void mqtt_unsubscribe_topic(const char *topic)
{
    if (mqtt_client == NULL || !mqtt_connected) {
        ESP_LOGW(TAG, "MQTT 客户端未连接，无法取消订阅主题");
        return;
    }
    
    int msg_id = esp_mqtt_client_unsubscribe(mqtt_client, topic);
    ESP_LOGI(TAG, "取消订阅主题 %s 成功，msg_id=%d", topic, msg_id);
}

// 示例：订阅命令主题
void subscribe_command_topics(void)
{
    mqtt_subscribe_topic("device/commands", 1);  // 使用 QoS 1
    mqtt_subscribe_topic("device/config", 1);    // 使用 QoS 1
}
```

### 3. 处理接收到的消息

```c
// 处理接收到的消息
static void handle_incoming_message(const char *topic, int topic_len, const char *data, int data_len)
{
    // 创建临时缓冲区存储主题和数据（添加结束符）
    char *topic_str = malloc(topic_len + 1);
    char *data_str = malloc(data_len + 1);
    
    if (topic_str == NULL || data_str == NULL) {
        ESP_LOGE(TAG, "内存分配失败");
        free(topic_str);
        free(data_str);
        return;
    }
    
    memcpy(topic_str, topic, topic_len);
    memcpy(data_str, data, data_len);
    topic_str[topic_len] = '\0';
    data_str[data_len] = '\0';
    
    ESP_LOGI(TAG, "收到消息：主题=%s，数据=%s", topic_str, data_str);
    
    // 根据主题处理不同的消息
    if (strcmp(topic_str, "device/commands") == 0) {
        // 处理命令消息
        if (strcmp(data_str, "reboot") == 0) {
            ESP_LOGI(TAG, "收到重启命令，准备重启...");
            vTaskDelay(pdMS_TO_TICKS(1000));
            esp_restart();
        } else if (strcmp(data_str, "led_on") == 0) {
            ESP_LOGI(TAG, "收到开灯命令");
            // 这里添加开灯代码
        } else if (strcmp(data_str, "led_off") == 0) {
            ESP_LOGI(TAG, "收到关灯命令");
            // 这里添加关灯代码
        }
    } else if (strcmp(topic_str, "device/config") == 0) {
        // 处理配置消息
        ESP_LOGI(TAG, "收到配置消息：%s", data_str);
        // 这里可以解析 JSON 配置并应用
    }
    
    free(topic_str);
    free(data_str);
}

// 修改 MQTT 事件处理函数中的 MQTT_EVENT_DATA 部分
case MQTT_EVENT_DATA:
    ESP_LOGI(TAG, "MQTT_EVENT_DATA");
    handle_incoming_message(event->topic, event->topic_len, event->data, event->data_len);
    break;
```

## 高级用法

### 1. 使用 TLS/SSL 安全连接

```c
void mqtt_app_start_secure(void)
{
    // MQTT 安全客户端配置
    esp_mqtt_client_config_t mqtt_cfg = {
        .broker.address.uri = "mqtts://mqtt.eclipseprojects.io:8883",  // 使用 SSL/TLS
        .broker.verification.certificate = (const char *)mqtt_eclipseprojects_io_pem_start,  // 服务器证书
        .broker.verification.skip_cert_common_name_check = false,  // 检查证书通用名称
        .credentials.username = "your_username",
        .credentials.authentication.password = "your_password",
        .session.keepalive = 120,
        .session.disable_clean_session = false,
        .network.reconnect_timeout_ms = 10000,
        .network.transport = MQTT_TRANSPORT_OVER_SSL,  // 使用 SSL 传输
    };
    
    // 创建 MQTT 客户端
    mqtt_client = esp_mqtt_client_init(&mqtt_cfg);
    
    // 注册 MQTT 事件处理函数
    esp_mqtt_client_register_event(mqtt_client, ESP_EVENT_ANY_ID, mqtt_event_handler, NULL);
    
    // 启动 MQTT 客户端
    esp_mqtt_client_start(mqtt_client);
}
```

### 2. 使用遗嘱消息

```c
void mqtt_app_start_with_lwt(void)
{
    // MQTT 客户端配置（带遗嘱消息）
    esp_mqtt_client_config_t mqtt_cfg = {
        .broker.address.uri = "mqtt://mqtt.eclipseprojects.io",
        .broker.address.port = 1883,
        .credentials.username = NULL,
        .credentials.authentication.password = NULL,
        .session.keepalive = 120,
        .session.disable_clean_session = false,
        .session.last_will.topic = "device/status",  // 遗嘱主题
        .session.last_will.msg = "offline",          // 遗嘱消息
        .session.last_will.qos = 1,                  // 遗嘱 QoS
        .session.last_will.retain = 1,               // 保留遗嘱消息
    };
    
    // 创建 MQTT 客户端
    mqtt_client = esp_mqtt_client_init(&mqtt_cfg);
    
    // 注册 MQTT 事件处理函数
    esp_mqtt_client_register_event(mqtt_client, ESP_EVENT_ANY_ID, mqtt_event_handler, NULL);
    
    // 启动 MQTT 客户端
    esp_mqtt_client_start(mqtt_client);
    
    // 连接成功后发布在线状态
    if (mqtt_connected) {
        esp_mqtt_client_publish(mqtt_client, "device/status", "online", 0, 1, 1);
    }
}
```

### 3. 使用 MQTT 5.0 功能

ESP-IDF 最新版本支持 MQTT 5.0，提供了更多高级功能：

```c
#if ESP_IDF_VERSION >= ESP_IDF_VERSION_VAL(5, 0, 0)
void mqtt_app_start_v5(void)
{
    // MQTT 5.0 客户端配置
    esp_mqtt_client_config_t mqtt_cfg = {
        .broker.address.uri = "mqtt://mqtt.eclipseprojects.io",
        .broker.address.port = 1883,
        .credentials.username = NULL,
        .credentials.authentication.password = NULL,
        .session.keepalive = 120,
        .session.disable_clean_session = false,
        .session.protocol_ver = MQTT_PROTOCOL_V_5,  // 使用 MQTT 5.0
    };
    
    // 创建 MQTT 客户端
    mqtt_client = esp_mqtt_client_init(&mqtt_cfg);
    
    // 注册 MQTT 事件处理函数
    esp_mqtt_client_register_event(mqtt_client, ESP_EVENT_ANY_ID, mqtt_event_handler, NULL);
    
    // 启动 MQTT 客户端
    esp_mqtt_client_start(mqtt_client);
}
#endif
```

## 常见问题

### 1. 连接失败

如果 MQTT 连接失败，可能的原因和解决方法：

- **网络问题**：
  - 确保 WiFi 已连接并能访问互联网
  - 检查 MQTT 服务器地址和端口是否正确
  - 确认防火墙未阻止 MQTT 流量

- **认证问题**：
  - 检查用户名和密码是否正确
  - 确认客户端 ID 是否符合服务器要求
  - 检查 TLS/SSL 证书是否有效

- **服务器限制**：
  - 确认服务器是否有连接数限制
  - 检查是否有 IP 地址限制
  - 确认服务器是否支持您使用的 MQTT 版本

### 2. 消息发布/订阅问题

如果消息发布或订阅出现问题，可能的原因和解决方法：

- **主题格式错误**：
  - 检查主题格式是否符合 MQTT 规范
  - 确认主题层级分隔符使用正确（/）

- **QoS 问题**：
  - 确认服务器支持您使用的 QoS 级别
  - 对于重要消息，使用 QoS 1 或 2

- **权限问题**：
  - 确认客户端有权限发布/订阅相关主题
  - 检查 ACL（访问控制列表）设置

### 3. 断线问题

如果 MQTT 客户端频繁断线，可能的原因和解决方法：

- **网络不稳定**：
  - 改善网络连接质量
  - 实现健壮的重连机制

- **保活设置不当**：
  - 调整 keepalive 时间
  - 确保在网络空闲时发送 PING 消息

- **服务器过载**：
  - 选择更可靠的 MQTT 服务器
  - 考虑使用商业 MQTT 服务

## 性能优化

### 1. 减少消息大小

```c
// 优化消息大小
void publish_optimized_sensor_data(float temperature, float humidity)
{
    // 使用简短的主题名
    const char *topic = "s/d";  // 而不是 "sensors/data"
    
    // 使用紧凑的消息格式
    char data[20];
    snprintf(data, sizeof(data), "%.1f,%.1f", temperature, humidity);
    
    mqtt_publish_message(topic, data, 1);
}
```

### 2. 批量处理消息

```c
// 批量发送消息
void publish_batch_sensor_data(float *temperatures, float *humidities, int count)
{
    char data[512];
    int offset = 0;
    
    // 构建批量数据
    offset += snprintf(data + offset, sizeof(data) - offset, "[");
    for (int i = 0; i < count; i++) {
        offset += snprintf(data + offset, sizeof(data) - offset, 
                          "{\"t\":%.1f,\"h\":%.1f}", temperatures[i], humidities[i]);
        
        if (i < count - 1) {
            offset += snprintf(data + offset, sizeof(data) - offset, ",");
        }
    }
    offset += snprintf(data + offset, sizeof(data) - offset, "]");
    
    mqtt_publish_message("sensors/batch", data, 1);
}
```

### 3. 优化 QoS 级别

```c
// 根据消息重要性选择 QoS
void publish_data_with_appropriate_qos(const char *topic, const char *data, bool is_critical)
{
    int qos = is_critical ? 1 : 0;  // 重要消息使用 QoS 1，普通消息使用 QoS 0
    mqtt_publish_message(topic, data, qos);
}
```

## 总结

本教程详细介绍了如何在 ESP-IDF 中使用 MQTT 客户端，包括：

1. **基本使用**：初始化、连接、事件处理
2. **重连机制**：基本重连和指数退避重连
3. **消息收发**：发布和订阅消息、处理接收到的消息
4. **高级功能**：TLS/SSL 安全连接、遗嘱消息、MQTT 5.0 支持
5. **性能优化**：减少消息大小、批量处理、优化 QoS 级别

通过实现健壮的重连机制和合理的消息处理策略，可以确保 MQTT 客户端在不稳定的网络环境中也能可靠工作。

希望本教程能帮助您在 ESP32 项目中成功集成 MQTT 功能。如有任何问题，欢迎在 Espressif 论坛或 GitHub 上提问。

