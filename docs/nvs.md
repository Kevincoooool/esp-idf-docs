


          
# ESP-IDF NVS（非易失性存储）使用教程

## 目录
- [NVS 简介](#nvs-简介)
- [基本使用](#基本使用)
- [加密存储](#加密存储)
- [高级用法](#高级用法)
- [常见问题](#常见问题)

## NVS 简介

NVS（Non-volatile Storage，非易失性存储）是 ESP-IDF 提供的一个键值对存储系统，可以用来存储配置信息、校准数据等需要掉电保存的数据。

### 特点：
- 键值对存储方式
- 支持多种数据类型
- 支持加密存储
- 支持命名空间隔离
- 掉电数据不丢失

## 基本使用

### 1. 初始化 NVS

```c
#include "nvs_flash.h"
#include "esp_log.h"

static const char *TAG = "nvs_example";

void app_main(void)
{
    // 初始化 NVS
    esp_err_t ret = nvs_flash_init();
    if (ret == ESP_ERR_NVS_NO_FREE_PAGES || ret == ESP_ERR_NVS_NEW_VERSION_FOUND) {
        // NVS 分区被擦除或格式化
        ESP_ERROR_CHECK(nvs_flash_erase());
        ret = nvs_flash_init();
    }
    ESP_ERROR_CHECK(ret);
    
    ESP_LOGI(TAG, "NVS 初始化成功");
}
```

### 2. 基本读写操作

```c
void nvs_basic_example(void)
{
    // 打开 NVS 句柄
    nvs_handle_t my_handle;
    esp_err_t ret = nvs_open("storage", NVS_READWRITE, &my_handle);
    if (ret != ESP_OK) {
        ESP_LOGE(TAG, "打开 NVS 句柄失败");
        return;
    }
    
    // 写入整数
    int32_t number = 42;
    ret = nvs_set_i32(my_handle, "number", number);
    if (ret != ESP_OK) {
        ESP_LOGE(TAG, "写入整数失败");
    }
    
    // 写入字符串
    char *str = "Hello NVS";
    ret = nvs_set_str(my_handle, "string", str);
    if (ret != ESP_OK) {
        ESP_LOGE(TAG, "写入字符串失败");
    }
    
    // 提交更改
    ret = nvs_commit(my_handle);
    if (ret != ESP_OK) {
        ESP_LOGE(TAG, "提交更改失败");
    }
    
    // 读取整数
    int32_t read_number;
    ret = nvs_get_i32(my_handle, "number", &read_number);
    if (ret == ESP_OK) {
        ESP_LOGI(TAG, "读取的整数: %d", read_number);
    }
    
    // 读取字符串
    size_t required_size;
    ret = nvs_get_str(my_handle, "string", NULL, &required_size);
    if (ret == ESP_OK) {
        char *read_str = malloc(required_size);
        ret = nvs_get_str(my_handle, "string", read_str, &required_size);
        if (ret == ESP_OK) {
            ESP_LOGI(TAG, "读取的字符串: %s", read_str);
        }
        free(read_str);
    }
    
    // 关闭句柄
    nvs_close(my_handle);
}
```

### 3. 存储不同类型的数据

```c
void nvs_types_example(void)
{
    nvs_handle_t handle;
    esp_err_t ret = nvs_open("storage", NVS_READWRITE, &handle);
    
    // 存储 8 位整数
    int8_t i8_value = 123;
    nvs_set_i8(handle, "i8_key", i8_value);
    
    // 存储 16 位整数
    int16_t i16_value = 12345;
    nvs_set_i16(handle, "i16_key", i16_value);
    
    // 存储 32 位整数
    int32_t i32_value = 1234567;
    nvs_set_i32(handle, "i32_key", i32_value);
    
    // 存储 64 位整数
    int64_t i64_value = 1234567890123;
    nvs_set_i64(handle, "i64_key", i64_value);
    
    // 存储二进制数据（Blob）
    uint8_t data[] = {1, 2, 3, 4, 5};
    nvs_set_blob(handle, "blob_key", data, sizeof(data));
    
    // 提交更改
    nvs_commit(handle);
    
    // 读取示例
    int8_t i8_read;
    nvs_get_i8(handle, "i8_key", &i8_read);
    ESP_LOGI(TAG, "读取 8 位整数: %d", i8_read);
    
    // 关闭句柄
    nvs_close(handle);
}
```

## 加密存储

### 1. 启用 NVS 加密

```c
#include "nvs_flash.h"
#include "nvs_sec_provider.h"

void nvs_encrypted_example(void)
{
    // 加密密钥（256 位）
    const uint8_t key[32] = {
        // 这里填入您的密钥
        0x11, 0x22, 0x33, 0x44, 0x55, 0x66, 0x77, 0x88,
        0x99, 0xAA, 0xBB, 0xCC, 0xDD, 0xEE, 0xFF, 0x00,
        0x11, 0x22, 0x33, 0x44, 0x55, 0x66, 0x77, 0x88,
        0x99, 0xAA, 0xBB, 0xCC, 0xDD, 0xEE, 0xFF, 0x00
    };
    
    // 初始化加密的 NVS 分区
    esp_err_t ret = nvs_flash_init_encrypted(key);
    if (ret == ESP_ERR_NVS_NO_FREE_PAGES || ret == ESP_ERR_NVS_NEW_VERSION_FOUND) {
        ESP_ERROR_CHECK(nvs_flash_erase());
        ret = nvs_flash_init_encrypted(key);
    }
    ESP_ERROR_CHECK(ret);
    
    // 使用加密的 NVS
    nvs_handle_t handle;
    ret = nvs_open("secure", NVS_READWRITE, &handle);
    if (ret != ESP_OK) {
        ESP_LOGE(TAG, "打开加密 NVS 失败");
        return;
    }
    
    // 存储敏感数据
    const char *secret = "sensitive_data";
    nvs_set_str(handle, "secret", secret);
    nvs_commit(handle);
    
    // 读取敏感数据
    size_t required_size;
    nvs_get_str(handle, "secret", NULL, &required_size);
    char *read_secret = malloc(required_size);
    nvs_get_str(handle, "secret", read_secret, &required_size);
    ESP_LOGI(TAG, "读取的敏感数据: %s", read_secret);
    
    free(read_secret);
    nvs_close(handle);
}
```

## 高级用法

### 1. 使用命名空间

```c
void nvs_namespace_example(void)
{
    // 打开不同的命名空间
    nvs_handle_t wifi_handle;
    nvs_handle_t bluetooth_handle;
    
    // WiFi 配置命名空间
    nvs_open("wifi", NVS_READWRITE, &wifi_handle);
    nvs_set_str(wifi_handle, "ssid", "my_wifi");
    nvs_set_str(wifi_handle, "password", "my_password");
    nvs_commit(wifi_handle);
    
    // 蓝牙配置命名空间
    nvs_open("bluetooth", NVS_READWRITE, &bluetooth_handle);
    nvs_set_str(bluetooth_handle, "name", "my_device");
    nvs_set_u8(bluetooth_handle, "power", 100);
    nvs_commit(bluetooth_handle);
    
    // 关闭句柄
    nvs_close(wifi_handle);
    nvs_close(bluetooth_handle);
}
```

### 2. 迭代所有键值

```c
void nvs_iterator_example(void)
{
    nvs_handle_t handle;
    nvs_open("storage", NVS_READONLY, &handle);
    
    // 获取使用的条目数
    size_t used_entries;
    nvs_get_used_entry_count(handle, &used_entries);
    ESP_LOGI(TAG, "NVS 中使用的条目数: %d", used_entries);
    
    // 遍历所有条目
    nvs_entry_info_t info;
    nvs_iterator_t it = nvs_entry_find("storage", NULL, NVS_TYPE_ANY);
    while (it != NULL) {
        nvs_entry_info(it, &info);
        ESP_LOGI(TAG, "键: %s, 类型: %d", info.key, info.type);
        it = nvs_entry_next(it);
    }
    nvs_release_iterator(it);
    
    nvs_close(handle);
}
```

## 常见问题

### 1. NVS 分区满了

如果遇到 NVS 分区空间不足，可以：
- 增加分区大小（修改分区表）
- 删除不需要的键值对
- 定期清理过期数据

### 2. 数据一致性

为确保数据一致性：
- 每次写入后调用 nvs_commit()
- 使用事务操作处理多个相关的写入
- 定期备份重要数据

### 3. 性能优化

提高 NVS 性能的建议：
- 减少写入频率
- 批量处理数据
- 合理使用命名空间
- 避免频繁擦除操作

## 总结

NVS 是 ESP-IDF 中一个强大的数据存储组件，通过本教程，您应该已经掌握了：
- NVS 的基本使用方法
- 不同数据类型的存储
- 加密存储的实现
- 高级功能的使用

建议在实际应用中根据需求选择合适的存储方式，合理使用命名空间，并注意数据安全性。

## 参考资料

- [ESP-IDF NVS 编程指南](https://docs.espressif.com/projects/esp-idf/zh_CN/latest/esp32/api-reference/storage/nvs_flash.html)
- [ESP32 技术参考手册](https://www.espressif.com/sites/default/files/documentation/esp32_technical_reference_manual_cn.pdf)

        