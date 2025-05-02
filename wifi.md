


          
# ESP-IDF WiFi 使用教程

## 目录
- [WiFi 简介](#wifi-简介)
- [基本使用](#基本使用)
- [WiFi 重连机制](#wifi-重连机制)
- [高级用法](#高级用法)
- [常见问题](#常见问题)
- [性能优化](#性能优化)

## WiFi 简介

ESP32/ESP32-S3 内置 WiFi 功能，支持 802.11 b/g/n 标准，可以工作在 Station 模式（作为客户端连接到路由器）、SoftAP 模式（作为接入点供其他设备连接）或两者同时工作。ESP-IDF 提供了完整的 WiFi 驱动和 API，使得 WiFi 功能的使用变得简单。

## 基本使用

### 1. WiFi 初始化

```c
#include <string.h>
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "freertos/event_groups.h"
#include "esp_system.h"
#include "esp_wifi.h"
#include "esp_event.h"
#include "esp_log.h"
#include "nvs_flash.h"

#include "lwip/err.h"
#include "lwip/sys.h"

#define WIFI_SSID      "你的WiFi名称"
#define WIFI_PASS      "你的WiFi密码"
#define MAXIMUM_RETRY  5

static const char *TAG = "wifi_example";

/* FreeRTOS 事件组，用于WiFi连接状态同步 */
static EventGroupHandle_t s_wifi_event_group;

/* 事件组位定义 */
#define WIFI_CONNECTED_BIT BIT0
#define WIFI_FAIL_BIT      BIT1

static int s_retry_num = 0;

static void event_handler(void* arg, esp_event_base_t event_base,
                          int32_t event_id, void* event_data)
{
    if (event_base == WIFI_EVENT && event_id == WIFI_EVENT_STA_START) {
        esp_wifi_connect();
    } else if (event_base == WIFI_EVENT && event_id == WIFI_EVENT_STA_DISCONNECTED) {
        if (s_retry_num < MAXIMUM_RETRY) {
            esp_wifi_connect();
            s_retry_num++;
            ESP_LOGI(TAG, "重试连接到 AP");
        } else {
            xEventGroupSetBits(s_wifi_event_group, WIFI_FAIL_BIT);
        }
        ESP_LOGI(TAG,"连接到 AP 失败");
    } else if (event_base == IP_EVENT && event_id == IP_EVENT_STA_GOT_IP) {
        ip_event_got_ip_t* event = (ip_event_got_ip_t*) event_data;
        ESP_LOGI(TAG, "获取到 IP:" IPSTR, IP2STR(&event->ip_info.ip));
        s_retry_num = 0;
        xEventGroupSetBits(s_wifi_event_group, WIFI_CONNECTED_BIT);
    }
}

void wifi_init_sta(void)
{
    // 创建 WiFi 事件组
    s_wifi_event_group = xEventGroupCreate();

    // 初始化底层 TCP/IP 适配器
    ESP_ERROR_CHECK(esp_netif_init());

    // 创建默认事件循环
    ESP_ERROR_CHECK(esp_event_loop_create_default());
    
    // 创建默认 WiFi station 接口
    esp_netif_create_default_wifi_sta();

    // 初始化 WiFi 
    wifi_init_config_t cfg = WIFI_INIT_CONFIG_DEFAULT();
    ESP_ERROR_CHECK(esp_wifi_init(&cfg));

    // 注册事件处理函数
    ESP_ERROR_CHECK(esp_event_handler_register(WIFI_EVENT, ESP_EVENT_ANY_ID, &event_handler, NULL));
    ESP_ERROR_CHECK(esp_event_handler_register(IP_EVENT, IP_EVENT_STA_GOT_IP, &event_handler, NULL));

    // 配置 WiFi 连接参数
    wifi_config_t wifi_config = {
        .sta = {
            .ssid = WIFI_SSID,
            .password = WIFI_PASS,
            /* 设置认证模式为 WPA2-PSK
             * 如果设置为 WIFI_AUTH_OPEN，则表示连接开放网络 */
            .threshold.authmode = WIFI_AUTH_WPA2_PSK,
            .pmf_cfg = {
                .capable = true,
                .required = false
            },
        },
    };
    
    // 设置 WiFi 工作模式为 Station 模式
    ESP_ERROR_CHECK(esp_wifi_set_mode(WIFI_MODE_STA));
    
    // 设置 WiFi 配置
    ESP_ERROR_CHECK(esp_wifi_set_config(WIFI_IF_STA, &wifi_config));
    
    // 启动 WiFi
    ESP_ERROR_CHECK(esp_wifi_start());

    ESP_LOGI(TAG, "wifi_init_sta 完成");

    /* 等待连接建立或超过最大重试次数 */
    EventBits_t bits = xEventGroupWaitBits(s_wifi_event_group,
            WIFI_CONNECTED_BIT | WIFI_FAIL_BIT,
            pdFALSE,
            pdFALSE,
            portMAX_DELAY);

    /* 处理连接结果 */
    if (bits & WIFI_CONNECTED_BIT) {
        ESP_LOGI(TAG, "已连接到 SSID:%s", WIFI_SSID);
    } else if (bits & WIFI_FAIL_BIT) {
        ESP_LOGI(TAG, "连接到 SSID:%s 失败", WIFI_SSID);
    } else {
        ESP_LOGE(TAG, "意外事件");
    }

    // 注销事件处理函数（如果不再需要）
    // ESP_ERROR_CHECK(esp_event_handler_unregister(IP_EVENT, IP_EVENT_STA_GOT_IP, &event_handler));
    // ESP_ERROR_CHECK(esp_event_handler_unregister(WIFI_EVENT, ESP_EVENT_ANY_ID, &event_handler));
    // vEventGroupDelete(s_wifi_event_group);
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

    ESP_LOGI(TAG, "ESP_WIFI_MODE_STA");
    wifi_init_sta();
}
```

### 2. SoftAP 模式

```c
#define WIFI_AP_SSID      "ESP32_AP"
#define WIFI_AP_PASS      "password"
#define WIFI_AP_CHANNEL   1
#define WIFI_AP_MAX_STA   4

void wifi_init_softap(void)
{
    // 初始化 TCP/IP 适配器
    ESP_ERROR_CHECK(esp_netif_init());
    
    // 创建默认事件循环
    ESP_ERROR_CHECK(esp_event_loop_create_default());
    
    // 创建默认 WiFi AP 接口
    esp_netif_create_default_wifi_ap();

    // 初始化 WiFi
    wifi_init_config_t cfg = WIFI_INIT_CONFIG_DEFAULT();
    ESP_ERROR_CHECK(esp_wifi_init(&cfg));

    // 注册事件处理函数
    ESP_ERROR_CHECK(esp_event_handler_register(WIFI_EVENT, ESP_EVENT_ANY_ID, &wifi_event_handler, NULL));

    // 配置 SoftAP
    wifi_config_t wifi_config = {
        .ap = {
            .ssid = WIFI_AP_SSID,
            .ssid_len = strlen(WIFI_AP_SSID),
            .channel = WIFI_AP_CHANNEL,
            .password = WIFI_AP_PASS,
            .max_connection = WIFI_AP_MAX_STA,
            .authmode = WIFI_AUTH_WPA_WPA2_PSK
        },
    };
    
    // 如果密码长度为0，则设置为开放网络
    if (strlen(WIFI_AP_PASS) == 0) {
        wifi_config.ap.authmode = WIFI_AUTH_OPEN;
    }

    // 设置 WiFi 工作模式为 AP 模式
    ESP_ERROR_CHECK(esp_wifi_set_mode(WIFI_MODE_AP));
    
    // 设置 AP 配置
    ESP_ERROR_CHECK(esp_wifi_set_config(WIFI_IF_AP, &wifi_config));
    
    // 启动 WiFi
    ESP_ERROR_CHECK(esp_wifi_start());

    ESP_LOGI(TAG, "wifi_init_softap 完成，SSID:%s password:%s channel:%d",
             WIFI_AP_SSID, WIFI_AP_PASS, WIFI_AP_CHANNEL);
}
```

## WiFi 重连机制

### 1. 基本重连机制

```c
#define WIFI_RECONNECT_INTERVAL  5000  // 重连间隔，单位毫秒

static bool s_is_wifi_connected = false;
static bool s_reconnect_enabled = true;
static TimerHandle_t s_wifi_reconnect_timer = NULL;

// WiFi 重连定时器回调函数
static void wifi_reconnect_timer_callback(TimerHandle_t xTimer)
{
    if (s_reconnect_enabled && !s_is_wifi_connected) {
        ESP_LOGI(TAG, "尝试重新连接到 WiFi...");
        esp_wifi_connect();
    }
}

// WiFi 事件处理函数
static void wifi_event_handler(void* arg, esp_event_base_t event_base,
                               int32_t event_id, void* event_data)
{
    if (event_base == WIFI_EVENT) {
        if (event_id == WIFI_EVENT_STA_START) {
            // WiFi 启动后尝试连接
            esp_wifi_connect();
        } else if (event_id == WIFI_EVENT_STA_CONNECTED) {
            ESP_LOGI(TAG, "已连接到 AP");
        } else if (event_id == WIFI_EVENT_STA_DISCONNECTED) {
            wifi_event_sta_disconnected_t* event = (wifi_event_sta_disconnected_t*) event_data;
            ESP_LOGW(TAG, "WiFi 断开连接，原因: %d", event->reason);
            
            s_is_wifi_connected = false;
            
            // 启动重连定时器
            if (s_reconnect_enabled) {
                xTimerStart(s_wifi_reconnect_timer, 0);
            }
        }
    } else if (event_base == IP_EVENT) {
        if (event_id == IP_EVENT_STA_GOT_IP) {
            ip_event_got_ip_t* event = (ip_event_got_ip_t*) event_data;
            ESP_LOGI(TAG, "获取到 IP:" IPSTR, IP2STR(&event->ip_info.ip));
            
            s_is_wifi_connected = true;
            
            // 停止重连定时器
            xTimerStop(s_wifi_reconnect_timer, 0);
        } else if (event_id == IP_EVENT_STA_LOST_IP) {
            ESP_LOGW(TAG, "失去 IP 地址");
            s_is_wifi_connected = false;
        }
    }
}

// 初始化 WiFi 重连机制
void init_wifi_reconnect(void)
{
    // 创建重连定时器
    s_wifi_reconnect_timer = xTimerCreate(
        "wifi_reconnect_timer",
        pdMS_TO_TICKS(WIFI_RECONNECT_INTERVAL),
        pdTRUE,  // 自动重载
        NULL,
        wifi_reconnect_timer_callback);
    
    // 注册事件处理函数
    ESP_ERROR_CHECK(esp_event_handler_register(WIFI_EVENT, ESP_EVENT_ANY_ID, &wifi_event_handler, NULL));
    ESP_ERROR_CHECK(esp_event_handler_register(IP_EVENT, ESP_EVENT_ANY_ID, &wifi_event_handler, NULL));
}

// 启用/禁用 WiFi 重连
void enable_wifi_reconnect(bool enable)
{
    s_reconnect_enabled = enable;
    
    if (!enable) {
        xTimerStop(s_wifi_reconnect_timer, 0);
    } else if (!s_is_wifi_connected) {
        xTimerStart(s_wifi_reconnect_timer, 0);
    }
}
```

### 2. 高级重连机制（指数退避）

```c
#define WIFI_RECONNECT_INTERVAL_MIN  1000   // 最小重连间隔，单位毫秒
#define WIFI_RECONNECT_INTERVAL_MAX  30000  // 最大重连间隔，单位毫秒
#define WIFI_RECONNECT_FACTOR        2      // 退避因子

static uint32_t s_current_reconnect_interval = WIFI_RECONNECT_INTERVAL_MIN;

// WiFi 重连定时器回调函数（指数退避）
static void wifi_reconnect_timer_callback_backoff(TimerHandle_t xTimer)
{
    if (s_reconnect_enabled && !s_is_wifi_connected) {
        ESP_LOGI(TAG, "尝试重新连接到 WiFi...");
        esp_wifi_connect();
        
        // 更新下一次重连间隔（指数退避）
        s_current_reconnect_interval *= WIFI_RECONNECT_FACTOR;
        if (s_current_reconnect_interval > WIFI_RECONNECT_INTERVAL_MAX) {
            s_current_reconnect_interval = WIFI_RECONNECT_INTERVAL_MAX;
        }
        
        // 更新定时器周期
        xTimerChangePeriod(s_wifi_reconnect_timer, 
                          pdMS_TO_TICKS(s_current_reconnect_interval), 0);
    }
}

// 在连接成功时重置重连间隔
static void reset_reconnect_interval(void)
{
    s_current_reconnect_interval = WIFI_RECONNECT_INTERVAL_MIN;
}
```

## 高级用法

### 1. 同时使用 Station 和 SoftAP 模式

```c
void wifi_init_sta_ap(void)
{
    // 初始化 TCP/IP 适配器
    ESP_ERROR_CHECK(esp_netif_init());
    
    // 创建默认事件循环
    ESP_ERROR_CHECK(esp_event_loop_create_default());
    
    // 创建默认 WiFi 接口
    esp_netif_create_default_wifi_sta();
    esp_netif_create_default_wifi_ap();

    // 初始化 WiFi
    wifi_init_config_t cfg = WIFI_INIT_CONFIG_DEFAULT();
    ESP_ERROR_CHECK(esp_wifi_init(&cfg));

    // 注册事件处理函数
    ESP_ERROR_CHECK(esp_event_handler_register(WIFI_EVENT, ESP_EVENT_ANY_ID, &wifi_event_handler, NULL));
    ESP_ERROR_CHECK(esp_event_handler_register(IP_EVENT, IP_EVENT_STA_GOT_IP, &wifi_event_handler, NULL));

    // 配置 Station 模式
    wifi_config_t sta_config = {
        .sta = {
            .ssid = WIFI_SSID,
            .password = WIFI_PASS,
            .threshold.authmode = WIFI_AUTH_WPA2_PSK,
        },
    };
    
    // 配置 SoftAP 模式
    wifi_config_t ap_config = {
        .ap = {
            .ssid = WIFI_AP_SSID,
            .ssid_len = strlen(WIFI_AP_SSID),
            .password = WIFI_AP_PASS,
            .channel = WIFI_AP_CHANNEL,
            .max_connection = WIFI_AP_MAX_STA,
            .authmode = WIFI_AUTH_WPA_WPA2_PSK
        },
    };

    // 设置 WiFi 工作模式为 Station + AP 模式
    ESP_ERROR_CHECK(esp_wifi_set_mode(WIFI_MODE_APSTA));
    
    // 设置 Station 和 AP 配置
    ESP_ERROR_CHECK(esp_wifi_set_config(WIFI_IF_STA, &sta_config));
    ESP_ERROR_CHECK(esp_wifi_set_config(WIFI_IF_AP, &ap_config));
    
    // 启动 WiFi
    ESP_ERROR_CHECK(esp_wifi_start());

    ESP_LOGI(TAG, "WiFi 初始化完成，同时工作在 Station 和 SoftAP 模式");
}
```

### 2. WiFi 扫描

```c
#define DEFAULT_SCAN_LIST_SIZE 10

static void wifi_scan(void)
{
    uint16_t number = DEFAULT_SCAN_LIST_SIZE;
    wifi_ap_record_t ap_info[DEFAULT_SCAN_LIST_SIZE];
    uint16_t ap_count = 0;
    memset(ap_info, 0, sizeof(ap_info));

    // 设置扫描配置
    wifi_scan_config_t scan_config = {
        .ssid = NULL,
        .bssid = NULL,
        .channel = 0,
        .show_hidden = true
    };

    // 开始扫描
    ESP_ERROR_CHECK(esp_wifi_scan_start(&scan_config, true));
    
    // 获取扫描结果
    ESP_ERROR_CHECK(esp_wifi_scan_get_ap_records(&number, ap_info));
    ESP_ERROR_CHECK(esp_wifi_scan_get_ap_num(&ap_count));
    
    ESP_LOGI(TAG, "扫描完成，共找到 %u 个接入点:", ap_count);
    
    for (int i = 0; i < ap_count; i++) {
        ESP_LOGI(TAG, "SSID: %s, RSSI: %d, Channel: %d, Auth mode: %d", 
                 ap_info[i].ssid, ap_info[i].rssi, ap_info[i].primary, ap_info[i].authmode);
    }
}
```

### 3. 使用 WPS 连接

```c
static void wifi_wps_event_handler(void* arg, esp_event_base_t event_base,
                                  int32_t event_id, void* event_data)
{
    if (event_base == WIFI_EVENT && event_id == WIFI_EVENT_STA_WPS_ER_SUCCESS) {
        ESP_LOGI(TAG, "WPS 成功，正在连接到 AP...");
        esp_wifi_connect();
    } else if (event_base == WIFI_EVENT && event_id == WIFI_EVENT_STA_WPS_ER_FAILED) {
        ESP_LOGI(TAG, "WPS 失败，重试...");
        esp_wifi_wps_disable();
        esp_wifi_wps_enable(NULL);
        esp_wifi_wps_start(0);
    } else if (event_base == WIFI_EVENT && event_id == WIFI_EVENT_STA_WPS_ER_TIMEOUT) {
        ESP_LOGI(TAG, "WPS 超时，重试...");
        esp_wifi_wps_disable();
        esp_wifi_wps_enable(NULL);
        esp_wifi_wps_start(0);
    }
}

void wifi_wps_start(void)
{
    // 注册 WPS 事件处理函数
    ESP_ERROR_CHECK(esp_event_handler_register(WIFI_EVENT, ESP_EVENT_ANY_ID, &wifi_wps_event_handler, NULL));
    
    // 配置 WPS
    esp_wps_config_t wps_config = WPS_CONFIG_INIT_DEFAULT(WPS_TYPE_PBC);
    
    // 启用 WPS
    ESP_ERROR_CHECK(esp_wifi_wps_enable(&wps_config));
    
    // 开始 WPS
    ESP_ERROR_CHECK(esp_wifi_wps_start(0));
    
    ESP_LOGI(TAG, "WPS 模式启动，请按下路由器上的 WPS 按钮...");
}
```

## 常见问题

### 1. 连接失败

如果 WiFi 连接失败，可能的原因和解决方法：

- **密码错误**：
  - 检查 SSID 和密码是否正确
  - 确认认证模式设置正确（WPA/WPA2/WPA3）

- **信号弱**：
  - 减小与路由器的距离
  - 使用外部天线
  - 检查是否有干扰源

- **路由器问题**：
  - 重启路由器
  - 检查路由器是否限制了连接数
  - 检查路由器是否启用了 MAC 地址过滤

### 2. 连接不稳定

如果 WiFi 连接不稳定，可能的原因和解决方法：

- **电源问题**：
  - 确保供电稳定，电流足够
  - 使用滤波电容稳定电源

- **干扰问题**：
  - 更换 WiFi 信道
  - 远离其他 2.4GHz 设备
  - 考虑使用 5GHz WiFi（如果支持）

- **软件问题**：
  - 实现健壮的重连机制
  - 定期检查连接状态
  - 更新到最新的 ESP-IDF 版本

### 3. 功耗问题

如果 WiFi 功耗过高，可能的解决方法：

- **使用省电模式**：
  - 配置 WiFi Modem Sleep 模式
  - 使用 Light Sleep 或 Deep Sleep 模式

- **优化连接策略**：
  - 减少不必要的数据传输
  - 使用批量传输代替频繁小数据传输
  - 在不需要时禁用 WiFi

## 性能优化

### 1. 省电模式配置

```c
void wifi_power_save(void)
{
    // 配置 WiFi 省电模式
    ESP_ERROR_CHECK(esp_wifi_set_ps(WIFI_PS_MIN_MODEM));
    
    // 或者禁用省电模式以获得最佳性能
    // ESP_ERROR_CHECK(esp_wifi_set_ps(WIFI_PS_NONE));
    
    ESP_LOGI(TAG, "WiFi 省电模式已配置");
}
```

### 2. 优化 WiFi 配置

```c
void wifi_optimize_performance(void)
{
    // 设置 WiFi 国家/地区信息
    wifi_country_t country = {
        .cc = "CN",  // 国家代码
        .schan = 1,  // 起始信道
        .nchan = 13, // 信道数量
        .policy = WIFI_COUNTRY_POLICY_AUTO,
    };
    ESP_ERROR_CHECK(esp_wifi_set_country(&country));
    
    // 设置 WiFi 协议模式（可以提高兼容性或性能）
    // ESP_ERROR_CHECK(esp_wifi_set_protocol(WIFI_IF_STA, WIFI_PROTOCOL_11B | WIFI_PROTOCOL_11G | WIFI_PROTOCOL_11N));
    
    // 设置 WiFi 带宽
    // ESP_ERROR_CHECK(esp_wifi_set_bandwidth(WIFI_IF_STA, WIFI_BW_HT20));
    
    // 设置 WiFi 发射功率
    // ESP_ERROR_CHECK(esp_wifi_set_max_tx_power(84));  // 范围: 8~84, 对应 2dBm~20dBm
    
    ESP_LOGI(TAG, "WiFi 性能优化配置已应用");
}
```

### 3. 使用静态 IP 地址

```c
void wifi_use_static_ip(void)
{
    // 获取默认的 netif 实例
    esp_netif_t *netif = esp_netif_get_handle_from_ifkey("WIFI_STA_DEF");
    
    // 禁用 DHCP 客户端
    ESP_ERROR_CHECK(esp_netif_dhcpc_stop(netif));
    
    // 配置静态 IP 地址
    esp_netif_ip_info_t ip_info;
    memset(&ip_info, 0, sizeof(esp_netif_ip_info_t));
    
    // 设置 IP 地址、网关和子网掩码
    IP4_ADDR(&ip_info.ip, 192, 168, 1, 100);
    IP4_ADDR(&ip_info.gw, 192, 168, 1, 1);
    IP4_ADDR(&ip_info.netmask, 255, 255, 255, 0);
    
    // 应用静态 IP 配置
    ESP_ERROR_CHECK(esp_netif_set_ip_info(netif, &ip_info));
    
    // 配置 DNS 服务器
    esp_netif_dns_info_t dns_info;
    IP4_ADDR(&dns_info.ip.u_addr.ip4, 8, 8, 8, 8);  // 使用 Google DNS
    ESP_ERROR_CHECK(esp_netif_set_dns_info(netif, ESP_NETIF_DNS_MAIN, &dns_info));
    
    ESP_LOGI(TAG, "静态 IP 配置已应用: " IPSTR, IP2STR(&ip_info.ip));
}
```

## 总结

ESP-IDF 提供了强大而灵活的 WiFi 功能，可以满足各种应用场景的需求。通过本教程，您应该已经掌握了：

- WiFi 的基本初始化和配置
- Station 和 SoftAP 模式的使用
- 健壮的 WiFi 重连机制实现
- 高级功能如扫描和 WPS
- 常见问题的解决方法
- 性能优化技巧

在实际应用中，应根据具体需求选择合适的 WiFi 工作模式和配置参数，并实现健壮的重连机制，以确保 WiFi 连接的稳定性和可靠性。

## 参考资料

- [ESP-IDF WiFi 编程指南](https://docs.espressif.com/projects/esp-idf/zh_CN/latest/esp32/api-guides/wifi.html)
- [ESP32 技术参考手册](https://www.espressif.com/sites/default/files/documentation/esp32_technical_reference_manual_cn.pdf)
- [ESP-IDF 示例代码](https://github.com/espressif/esp-idf/tree/master/examples/wifi)

