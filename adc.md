


          
# ESP-IDF ADC 使用教程

## 目录
- [简介](#简介)
- [ESP32-S3 ADC 基础知识](#esp32-s3-adc-基础知识)
- [ESP-IDF 中的 ADC 配置](#esp-idf-中的-adc-配置)
- [ADC 按键示例](#adc-按键示例)
- [常见问题与解决方案](#常见问题与解决方案)
- [高级应用](#高级应用)
- [总结](#总结)
- [参考资料](#参考资料)

## 简介

ADC（模数转换器）是 ESP32-S3 等微控制器中的重要外设，用于将模拟信号转换为数字信号。通过 ADC，我们可以读取各种模拟传感器的数据，如温度传感器、光敏电阻、电位器等。ESP32-S3 提供了两个 12 位 SAR ADC（逐次逼近型 ADC），支持多达 20 个通道。

本教程将详细介绍如何在 ESP-IDF 框架下使用 ADC 功能，并以 ADC 按键为例，展示 ADC 的初始化和读取操作。

## ESP32-S3 ADC 基础知识

### 1. ADC 特性

ESP32-S3 的 ADC 具有以下特性：

- 两个 12 位 SAR ADC（ADC1 和 ADC2）
- 支持多达 20 个通道
- 采样率高达 2 MHz
- 支持单次采样和连续采样模式
- 内置校准功能，提高测量精度
- 支持多种衰减选项，扩展输入电压范围

### 2. ADC 通道分配

ESP32-S3 的 ADC 通道与 GPIO 引脚的对应关系如下：

- **ADC1**：支持 10 个通道（ADC1_CH0 ~ ADC1_CH9），对应 GPIO1 ~ GPIO10
- **ADC2**：支持 10 个通道（ADC2_CH0 ~ ADC2_CH9），对应 GPIO11 ~ GPIO20

在本教程中，我们将使用 GPIO5（对应 ADC1_CH4）作为 ADC 按键输入。

### 3. ADC 衰减选项

ESP32-S3 的 ADC 支持多种衰减选项，用于扩展输入电压范围：

- **ADC_ATTEN_DB_0**：无衰减，输入范围 0 ~ 1.1V
- **ADC_ATTEN_DB_2_5**：2.5dB 衰减，输入范围 0 ~ 1.5V
- **ADC_ATTEN_DB_6**：6dB 衰减，输入范围 0 ~ 2.2V
- **ADC_ATTEN_DB_11**：11dB 衰减，输入范围 0 ~ 3.3V

## ESP-IDF 中的 ADC 配置

ESP-IDF 提供了两种使用 ADC 的方式：

1. **传统 API（adc_oneshot.h）**：适用于单次采样
2. **连续 API（adc_continuous.h）**：适用于连续采样

本教程将主要介绍传统 API 的使用方法，因为它更适合 ADC 按键等简单应用。

### 1. 包含必要的头文件

```c
#include "esp_adc/adc_oneshot.h"
#include "esp_adc/adc_cali.h"
#include "esp_adc/adc_cali_scheme.h"
```

### 2. ADC 配置步骤

在 ESP-IDF 中配置 ADC 通常遵循以下步骤：

1. 创建 ADC 单次采样句柄
2. 配置 ADC 通道和衰减
3. 读取 ADC 值
4. （可选）校准 ADC 值，转换为电压

## ADC 按键示例

下面是一个完整的示例，展示如何使用 ESP-IDF 的 ADC 功能实现 ADC 按键：

```c
#include <stdio.h>
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "esp_log.h"
#include "esp_adc/adc_oneshot.h"
#include "esp_adc/adc_cali.h"
#include "esp_adc/adc_cali_scheme.h"

static const char *TAG = "adc_example";

// 定义 ADC 通道（GPIO5 对应 ADC1_CH4）
#define ADC1_CHAN4 ADC_CHANNEL_4

// 定义按键阈值（根据实际电路调整）
#define KEY1_MIN_VAL 0
#define KEY1_MAX_VAL 1000
#define KEY2_MIN_VAL 1500
#define KEY2_MAX_VAL 2000
#define KEY3_MIN_VAL 2500
#define KEY3_MAX_VAL 3000

// ADC 校准句柄
static bool adc_calibration_init(adc_unit_t unit, adc_channel_t channel, adc_atten_t atten, adc_cali_handle_t *out_handle);
static void adc_calibration_deinit(adc_cali_handle_t handle);

void app_main(void)
{
    // 1. 创建 ADC 单次采样句柄
    adc_oneshot_unit_handle_t adc1_handle;
    adc_oneshot_unit_init_cfg_t init_config1 = {
        .unit_id = ADC_UNIT_1,
    };
    ESP_ERROR_CHECK(adc_oneshot_new_unit(&init_config1, &adc1_handle));
    
    // 2. 配置 ADC 通道
    adc_oneshot_chan_cfg_t config = {
        .bitwidth = ADC_BITWIDTH_DEFAULT,
        .atten = ADC_ATTEN_DB_11,  // 11dB 衰减，支持 0-3.3V 输入
    };
    ESP_ERROR_CHECK(adc_oneshot_config_channel(adc1_handle, ADC1_CHAN4, &config));
    
    // 3. 校准 ADC（可选）
    adc_cali_handle_t adc1_cali_handle = NULL;
    bool do_calibration = adc_calibration_init(ADC_UNIT_1, ADC1_CHAN4, ADC_ATTEN_DB_11, &adc1_cali_handle);
    
    ESP_LOGI(TAG, "ADC1 初始化完成，使用 GPIO5 作为输入");
    ESP_LOGI(TAG, "按键 1: %d-%d, 按键 2: %d-%d, 按键 3: %d-%d", 
             KEY1_MIN_VAL, KEY1_MAX_VAL, KEY2_MIN_VAL, KEY2_MAX_VAL, KEY3_MIN_VAL, KEY3_MAX_VAL);
    
    // 主循环
    while (1) {
        // 读取 ADC 值
        int adc_raw;
        ESP_ERROR_CHECK(adc_oneshot_read(adc1_handle, ADC1_CHAN4, &adc_raw));
        
        // 转换为电压值（如果已校准）
        int voltage = 0;
        if (do_calibration) {
            ESP_ERROR_CHECK(adc_cali_raw_to_voltage(adc1_cali_handle, adc_raw, &voltage));
            ESP_LOGI(TAG, "ADC 原始值: %d, 电压: %d mV", adc_raw, voltage);
        } else {
            ESP_LOGI(TAG, "ADC 原始值: %d", adc_raw);
        }
        
        // 判断按键
        if (adc_raw >= KEY1_MIN_VAL && adc_raw <= KEY1_MAX_VAL) {
            ESP_LOGI(TAG, "检测到按键 1 按下");
        } else if (adc_raw >= KEY2_MIN_VAL && adc_raw <= KEY2_MAX_VAL) {
            ESP_LOGI(TAG, "检测到按键 2 按下");
        } else if (adc_raw >= KEY3_MIN_VAL && adc_raw <= KEY3_MAX_VAL) {
            ESP_LOGI(TAG, "检测到按键 3 按下");
        }
        
        // 延时
        vTaskDelay(pdMS_TO_TICKS(300));
    }
    
    // 释放资源（实际上不会执行到这里）
    if (do_calibration) {
        adc_calibration_deinit(adc1_cali_handle);
    }
    ESP_ERROR_CHECK(adc_oneshot_del_unit(adc1_handle));
}

// ADC 校准初始化函数
static bool adc_calibration_init(adc_unit_t unit, adc_channel_t channel, adc_atten_t atten, adc_cali_handle_t *out_handle)
{
    adc_cali_handle_t handle = NULL;
    esp_err_t ret = ESP_FAIL;
    bool calibrated = false;
    
#if ADC_CALI_SCHEME_CURVE_FITTING_SUPPORTED
    // ESP32-S3 支持曲线拟合校准方案
    adc_cali_curve_fitting_config_t cali_config = {
        .unit_id = unit,
        .chan = channel,
        .atten = atten,
        .bitwidth = ADC_BITWIDTH_DEFAULT,
    };
    ret = adc_cali_create_scheme_curve_fitting(&cali_config, &handle);
    if (ret == ESP_OK) {
        calibrated = true;
    }
#endif

    *out_handle = handle;
    if (ret == ESP_OK) {
        ESP_LOGI(TAG, "校准方案: %s", "曲线拟合");
    } else {
        ESP_LOGW(TAG, "未使用校准方案");
    }
    
    return calibrated;
}

// ADC 校准释放函数
static void adc_calibration_deinit(adc_cali_handle_t handle)
{
#if ADC_CALI_SCHEME_CURVE_FITTING_SUPPORTED
    ESP_ERROR_CHECK(adc_cali_delete_scheme_curve_fitting(handle));
#endif
}
```

### ADC 按键电路设计

ADC 按键通常使用分压电路实现，如下图所示：

```
VCC (3.3V)
   |
   |
   R1 (10kΩ)
   |
   |------ GPIO5 (ADC 输入)
   |
   R2 (不同阻值，对应不同按键)
   |
   |
  GND
```

通过选择不同阻值的 R2，可以实现多个按键共用一个 ADC 通道。当按下不同按键时，ADC 读取到的值也不同，从而可以区分不同按键。

## 常见问题与解决方案

### 1. ADC 读数不稳定

如果 ADC 读数不稳定，可能的原因和解决方法：

- **电源噪声**：
  - 使用滤波电容稳定电源
  - 使用独立的模拟电源
  - 优化 PCB 布局，减少干扰

- **采样时间不足**：
  - 增加采样时间
  - 降低采样频率

- **输入阻抗过高**：
  - 使用低阻抗的信号源
  - 添加缓冲放大器

### 2. ADC 精度问题

如果 ADC 精度不满足要求，可能的解决方法：

- 使用校准功能提高精度
- 多次采样取平均值
- 使用外部高精度 ADC

### 3. ADC 按键误触发

如果 ADC 按键容易误触发，可能的解决方法：

- 调整按键阈值范围，增加容错空间
- 添加软件滤波，如多次采样确认
- 添加按键消抖处理

## 高级应用

### 1. 多通道 ADC 采样

ESP32-S3 支持多通道 ADC 采样，可以同时读取多个传感器的数据：

```c
// 配置多个 ADC 通道
ESP_ERROR_CHECK(adc_oneshot_config_channel(adc1_handle, ADC_CHANNEL_0, &config));
ESP_ERROR_CHECK(adc_oneshot_config_channel(adc1_handle, ADC_CHANNEL_3, &config));
ESP_ERROR_CHECK(adc_oneshot_config_channel(adc1_handle, ADC_CHANNEL_4, &config));

// 读取多个通道的 ADC 值
int adc_val_0, adc_val_3, adc_val_4;
ESP_ERROR_CHECK(adc_oneshot_read(adc1_handle, ADC_CHANNEL_0, &adc_val_0));
ESP_ERROR_CHECK(adc_oneshot_read(adc1_handle, ADC_CHANNEL_3, &adc_val_3));
ESP_ERROR_CHECK(adc_oneshot_read(adc1_handle, ADC_CHANNEL_4, &adc_val_4));
```

### 2. 连续 ADC 采样

对于需要高频采样的应用，可以使用连续 ADC 采样模式：

```c
#include "esp_adc/adc_continuous.h"

#define ADC_CONV_MODE ADC_CONV_SINGLE_UNIT_1  // 单 ADC 单元模式
#define ADC_OUTPUT_TYPE ADC_DIGI_OUTPUT_FORMAT_TYPE1  // 输出格式类型 1

static bool IRAM_ATTR s_conv_done_cb(adc_continuous_handle_t handle, const adc_continuous_evt_data_t *edata, void *user_data)
{
    BaseType_t high_task_awoken = pdFALSE;
    // 通知任务处理数据
    // ...
    return high_task_awoken;
}

void app_main(void)
{
    // 配置 ADC 连续采样
    adc_continuous_handle_t handle = NULL;
    adc_continuous_handle_cfg_t adc_config = {
        .max_store_buf_size = 1024,
        .conv_frame_size = 100,
    };
    ESP_ERROR_CHECK(adc_continuous_new_handle(&adc_config, &handle));
    
    // 配置 ADC 通道
    adc_continuous_config_t dig_cfg = {
        .pattern_num = 1,
        .adc_pattern = {{
            .atten = ADC_ATTEN_DB_11,
            .channel = ADC1_CHAN4,
            .unit = ADC_UNIT_1,
            .bit_width = ADC_BITWIDTH_DEFAULT,
        }},
        .conv_mode = ADC_CONV_MODE,
        .format = ADC_OUTPUT_TYPE,
    };
    ESP_ERROR_CHECK(adc_continuous_config(handle, &dig_cfg));
    
    // 注册回调函数
    adc_continuous_evt_cbs_t cbs = {
        .on_conv_done = s_conv_done_cb,
    };
    ESP_ERROR_CHECK(adc_continuous_register_event_callbacks(handle, &cbs, NULL));
    
    // 启动 ADC 连续采样
    ESP_ERROR_CHECK(adc_continuous_start(handle));
    
    // 主循环
    while (1) {
        // 处理 ADC 数据
        // ...
        vTaskDelay(pdMS_TO_TICKS(100));
    }
}
```

### 3. ADC 与 DMA 结合

ESP32-S3 支持使用 DMA 进行 ADC 数据传输，可以减少 CPU 负担：

```c
// 在 adc_continuous_config_t 中配置 DMA
adc_continuous_config_t dig_cfg = {
    // ... 其他配置 ...
    .conv_mode = ADC_CONV_MODE,
    .format = ADC_OUTPUT_TYPE,
};
ESP_ERROR_CHECK(adc_continuous_config(handle, &dig_cfg));

// 读取 DMA 缓冲区数据
uint8_t *buf = (uint8_t *)malloc(1024);
uint32_t ret_num = 0;
ESP_ERROR_CHECK(adc_continuous_read(handle, buf, 1024, &ret_num, 0));
```

## 总结

本教程详细介绍了如何在 ESP-IDF 框架下使用 ADC 功能，包括配置、基本操作以及 ADC 按键的完整示例。通过掌握这些知识，您可以轻松地在 ESP32-S3 项目中集成各种模拟传感器和 ADC 按键。

ESP32-S3 的 ADC 功能强大而灵活，支持多种采样模式和校准选项，可以满足大多数应用场景的需求。在实际应用中，应根据具体需求选择合适的 ADC 配置和采样方式，以获得最佳的性能和精度。

在后续的教程中，我们将介绍更多 ESP-IDF 的功能，如 DAC、触摸传感器等外设的使用方法。

## 参考资料

- [ESP-IDF 编程指南 - ADC 驱动程序](https://docs.espressif.com/projects/esp-idf/zh_CN/latest/esp32s3/api-reference/peripherals/adc.html)
- [ESP32-S3 技术规格书](https://www.espressif.com/sites/default/files/documentation/esp32-s3_datasheet_cn.pdf)
- [ADC 精度和校准](https://docs.espressif.com/projects/esp-idf/zh_CN/latest/esp32s3/api-reference/peripherals/adc.html#adc-accuracy)
