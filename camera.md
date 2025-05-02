


          
# ESP32-S3 驱动 OV2640 摄像头教程

## 目录
- [简介](#简介)
- [硬件准备](#硬件准备)
- [ESP-IDF 摄像头组件](#esp-idf-摄像头组件)
- [摄像头初始化](#摄像头初始化)
- [获取图像数据](#获取图像数据)
- [实用示例](#实用示例)
- [常见问题与解决方案](#常见问题与解决方案)
- [高级应用](#高级应用)
- [总结](#总结)
- [参考资料](#参考资料)

## 简介

ESP32-S3 是乐鑫推出的高性能 MCU，具有丰富的外设接口和强大的图像处理能力。OV2640 是一款常用的 200 万像素 CMOS 图像传感器，支持多种输出格式和分辨率。本教程将详细介绍如何在 ESP32-S3 Korvo2 v3 开发板上驱动 OV2640 摄像头，获取图像并实现显示或网页图传功能。

## 硬件准备

### ESP32-S3 Korvo2 v3 开发板

ESP32-S3 Korvo2 v3 是一款功能强大的开发板，具有以下特点：

- ESP32-S3 双核 Xtensa LX7 处理器，主频高达 240MHz
- 512KB SRAM 和 8MB PSRAM
- 16MB 闪存
- 支持 Wi-Fi 和蓝牙 5 (LE)
- 丰富的外设接口，包括摄像头接口

### OV2640 摄像头模块

OV2640 摄像头模块通常包含以下特性：

- 200 万像素 CMOS 传感器
- 最大分辨率：1600×1200
- 支持多种输出格式：YUV422/420、RGB565、JPEG 等
- 内置 ISP（图像信号处理器）
- 支持 SCCB（类似 I2C）接口配置

### 硬件连接

ESP32-S3 Korvo2 v3 开发板与 OV2640 摄像头模块的连接方式如下：

| OV2640 引脚 | ESP32-S3 引脚 | 说明 |
|------------|--------------|------|
| D0-D7      | GPIO15-GPIO22 | 8位并行数据线 |
| PCLK       | GPIO11       | 像素时钟 |
| VSYNC      | GPIO10       | 垂直同步 |
| HREF       | GPIO9        | 水平参考 |
| XCLK       | GPIO14       | 系统时钟 |
| SIOD       | GPIO4        | SCCB 数据线 |
| SIOC       | GPIO5        | SCCB 时钟线 |
| RESET      | GPIO12       | 复位 |
| PWDN       | GPIO13       | 掉电模式 |

注意：实际引脚可能因开发板版本而异，请参考开发板文档确认正确的引脚连接。

## ESP-IDF 摄像头组件

ESP-IDF 提供了 `esp_camera` 组件，简化了摄像头的驱动和操作。该组件支持多种摄像头型号，包括 OV2640、OV3660、OV5640 等。

### 安装 esp_camera 组件

首先，需要将 `esp_camera` 组件添加到项目中。有两种方法：

#### 方法一：使用 ESP-IDF 组件管理器

在项目的 `CMakeLists.txt` 文件中添加：

```cmake
set(EXTRA_COMPONENT_DIRS $ENV{IDF_PATH}/examples/common_components/protocol_examples_common)
```

然后在项目的 `main/CMakeLists.txt` 文件中添加：

```cmake
set(COMPONENT_REQUIRES esp_camera)
```

#### 方法二：手动添加组件

从 GitHub 克隆 `esp32-camera` 仓库到项目的 `components` 目录：

```bash
mkdir -p components
cd components
git clone https://github.com/espressif/esp32-camera.git
```

### esp_camera 组件 API 概览

`esp_camera` 组件提供了一系列 API 函数，用于摄像头的初始化、配置和图像获取：

#### 初始化与配置

- `esp_camera_init()`：初始化摄像头
- `esp_camera_deinit()`：反初始化摄像头
- `esp_camera_sensor_get()`：获取摄像头传感器对象

#### 图像获取

- `esp_camera_fb_get()`：获取一帧图像
- `esp_camera_fb_return()`：释放图像帧缓冲区

#### 参数设置

- `sensor->set_framesize()`：设置图像分辨率
- `sensor->set_quality()`：设置 JPEG 压缩质量
- `sensor->set_brightness()`：设置亮度
- `sensor->set_contrast()`：设置对比度
- `sensor->set_saturation()`：设置饱和度
- `sensor->set_special_effect()`：设置特效

## 摄像头初始化

### 配置摄像头参数

首先，需要定义摄像头配置参数：

```c
#include "esp_camera.h"
#include "esp_log.h"

static const char *TAG = "camera_example";

// 摄像头引脚定义
#define CAM_PIN_PWDN    13
#define CAM_PIN_RESET   12
#define CAM_PIN_XCLK    14
#define CAM_PIN_SIOD    4
#define CAM_PIN_SIOC    5

#define CAM_PIN_D7      22
#define CAM_PIN_D6      21
#define CAM_PIN_D5      20
#define CAM_PIN_D4      19
#define CAM_PIN_D3      18
#define CAM_PIN_D2      17
#define CAM_PIN_D1      16
#define CAM_PIN_D0      15
#define CAM_PIN_VSYNC   10
#define CAM_PIN_HREF    9
#define CAM_PIN_PCLK    11

static camera_config_t camera_config = {
    .pin_pwdn  = CAM_PIN_PWDN,
    .pin_reset = CAM_PIN_RESET,
    .pin_xclk = CAM_PIN_XCLK,
    .pin_sccb_sda = CAM_PIN_SIOD,
    .pin_sccb_scl = CAM_PIN_SIOC,

    .pin_d7 = CAM_PIN_D7,
    .pin_d6 = CAM_PIN_D6,
    .pin_d5 = CAM_PIN_D5,
    .pin_d4 = CAM_PIN_D4,
    .pin_d3 = CAM_PIN_D3,
    .pin_d2 = CAM_PIN_D2,
    .pin_d1 = CAM_PIN_D1,
    .pin_d0 = CAM_PIN_D0,
    .pin_vsync = CAM_PIN_VSYNC,
    .pin_href = CAM_PIN_HREF,
    .pin_pclk = CAM_PIN_PCLK,

    // XCLK 频率，单位为 Hz
    .xclk_freq_hz = 20000000,
    
    // 摄像头型号
    .ledc_timer = LEDC_TIMER_0,
    .ledc_channel = LEDC_CHANNEL_0,

    .pixel_format = PIXFORMAT_JPEG,  // 输出格式：JPEG
    .frame_size = FRAMESIZE_VGA,     // 分辨率：640x480
    
    .jpeg_quality = 12,              // JPEG 质量，0-63，数值越小质量越高
    .fb_count = 2,                   // 帧缓冲区数量
    .grab_mode = CAMERA_GRAB_WHEN_EMPTY,  // 抓取模式
};
```

### 初始化摄像头

使用上述配置参数初始化摄像头：

```c
esp_err_t camera_init() {
    // 初始化摄像头
    esp_err_t err = esp_camera_init(&camera_config);
    if (err != ESP_OK) {
        ESP_LOGE(TAG, "摄像头初始化失败 (0x%x)", err);
        return err;
    }
    
    // 获取摄像头传感器对象
    sensor_t * sensor = esp_camera_sensor_get();
    if (sensor == NULL) {
        ESP_LOGE(TAG, "获取摄像头传感器失败");
        return ESP_FAIL;
    }
    
    // 设置摄像头参数
    sensor->set_brightness(sensor, 0);     // -2 to 2
    sensor->set_contrast(sensor, 0);       // -2 to 2
    sensor->set_saturation(sensor, 0);     // -2 to 2
    sensor->set_special_effect(sensor, 0); // 0 = 无特效, 1 = 负片, 2 = 灰度, 3 = 偏红, 4 = 偏绿, 5 = 偏蓝, 6 = 复古
    
    sensor->set_whitebal(sensor, 1);       // 启用自动白平衡
    sensor->set_awb_gain(sensor, 1);       // 启用自动白平衡增益
    sensor->set_wb_mode(sensor, 0);        // 0 = 自动, 1 = 晴天, 2 = 阴天, 3 = 办公室, 4 = 家里, 5 = 夜晚
    
    sensor->set_exposure_ctrl(sensor, 1);  // 启用自动曝光
    sensor->set_aec2(sensor, 1);           // 启用自动曝光控制 2
    sensor->set_ae_level(sensor, 0);       // -2 to 2
    sensor->set_aec_value(sensor, 300);    // 0 to 1200
    
    sensor->set_gain_ctrl(sensor, 1);      // 启用自动增益控制
    sensor->set_agc_gain(sensor, 0);       // 0 to 30
    sensor->set_gainceiling(sensor, (gainceiling_t)0);  // 0 to 6
    
    sensor->set_hmirror(sensor, 0);        // 0 = 禁用, 1 = 启用水平镜像
    sensor->set_vflip(sensor, 0);          // 0 = 禁用, 1 = 启用垂直翻转
    
    ESP_LOGI(TAG, "摄像头初始化成功");
    return ESP_OK;
}
```

## 获取图像数据

### 获取单帧图像

使用 `esp_camera_fb_get()` 函数获取一帧图像：

```c
void get_camera_frame() {
    // 获取一帧图像
    camera_fb_t *fb = esp_camera_fb_get();
    if (!fb) {
        ESP_LOGE(TAG, "获取摄像头帧失败");
        return;
    }
    
    // 打印图像信息
    ESP_LOGI(TAG, "图像信息: 宽度=%d, 高度=%d, 格式=%d, 大小=%d 字节",
             fb->width, fb->height, fb->format, fb->len);
    
    // 在这里处理图像数据 (fb->buf, fb->len)
    // ...
    
    // 处理完毕后，释放帧缓冲区
    esp_camera_fb_return(fb);
}
```

### 连续获取图像

在实际应用中，通常需要连续获取图像。可以创建一个任务来实现：

```c
void camera_task(void *pvParameters) {
    while (1) {
        // 获取一帧图像
        camera_fb_t *fb = esp_camera_fb_get();
        if (!fb) {
            ESP_LOGE(TAG, "获取摄像头帧失败");
            vTaskDelay(pdMS_TO_TICKS(100));
            continue;
        }
        
        // 在这里处理图像数据 (fb->buf, fb->len)
        // ...
        
        // 处理完毕后，释放帧缓冲区
        esp_camera_fb_return(fb);
        
        // 短暂延时
        vTaskDelay(pdMS_TO_TICKS(50));
    }
}
```

## 实用示例

### 示例1：基本图像获取与保存

以下是一个完整的示例，展示如何初始化摄像头、获取图像并保存到 SD 卡：

```c
#include <stdio.h>
#include <string.h>
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "esp_camera.h"
#include "esp_log.h"
#include "esp_system.h"
#include "esp_vfs_fat.h"
#include "driver/sdmmc_host.h"
#include "driver/sdspi_host.h"
#include "sdmmc_cmd.h"

static const char *TAG = "camera_example";

// 摄像头引脚定义
#define CAM_PIN_PWDN    13
#define CAM_PIN_RESET   12
#define CAM_PIN_XCLK    14
#define CAM_PIN_SIOD    4
#define CAM_PIN_SIOC    5

#define CAM_PIN_D7      22
#define CAM_PIN_D6      21
#define CAM_PIN_D5      20
#define CAM_PIN_D4      19
#define CAM_PIN_D3      18
#define CAM_PIN_D2      17
#define CAM_PIN_D1      16
#define CAM_PIN_D0      15
#define CAM_PIN_VSYNC   10
#define CAM_PIN_HREF    9
#define CAM_PIN_PCLK    11

// SD 卡引脚定义
#define SD_PIN_MOSI     38
#define SD_PIN_MISO     37
#define SD_PIN_CLK      36
#define SD_PIN_CS       35

static camera_config_t camera_config = {
    .pin_pwdn  = CAM_PIN_PWDN,
    .pin_reset = CAM_PIN_RESET,
    .pin_xclk = CAM_PIN_XCLK,
    .pin_sccb_sda = CAM_PIN_SIOD,
    .pin_sccb_scl = CAM_PIN_SIOC,

    .pin_d7 = CAM_PIN_D7,
    .pin_d6 = CAM_PIN_D6,
    .pin_d5 = CAM_PIN_D5,
    .pin_d4 = CAM_PIN_D4,
    .pin_d3 = CAM_PIN_D3,
    .pin_d2 = CAM_PIN_D2,
    .pin_d1 = CAM_PIN_D1,
    .pin_d0 = CAM_PIN_D0,
    .pin_vsync = CAM_PIN_VSYNC,
    .pin_href = CAM_PIN_HREF,
    .pin_pclk = CAM_PIN_PCLK,

    .xclk_freq_hz = 20000000,
    .ledc_timer = LEDC_TIMER_0,
    .ledc_channel = LEDC_CHANNEL_0,

    .pixel_format = PIXFORMAT_JPEG,
    .frame_size = FRAMESIZE_VGA,
    
    .jpeg_quality = 12,
    .fb_count = 2,
    .grab_mode = CAMERA_GRAB_WHEN_EMPTY,
};

// 初始化 SD 卡
static esp_err_t init_sdcard(void)
{
    ESP_LOGI(TAG, "初始化 SD 卡");
    
    sdmmc_host_t host = SDSPI_HOST_DEFAULT();
    sdspi_slot_config_t slot_config = SDSPI_SLOT_CONFIG_DEFAULT();
    slot_config.gpio_miso = SD_PIN_MISO;
    slot_config.gpio_mosi = SD_PIN_MOSI;
    slot_config.gpio_sck  = SD_PIN_CLK;
    slot_config.gpio_cs   = SD_PIN_CS;
    
    esp_vfs_fat_sdmmc_mount_config_t mount_config = {
        .format_if_mount_failed = false,
        .max_files = 5,
        .allocation_unit_size = 16 * 1024
    };
    
    sdmmc_card_t *card;
    esp_err_t ret = esp_vfs_fat_sdmmc_mount("/sdcard", &host, &slot_config, &mount_config, &card);
    
    if (ret != ESP_OK) {
        if (ret == ESP_FAIL) {
            ESP_LOGE(TAG, "挂载 SD 卡失败");
        } else {
            ESP_LOGE(TAG, "初始化 SD 卡失败 (0x%x)", ret);
        }
        return ret;
    }
    
    ESP_LOGI(TAG, "SD 卡初始化成功");
    return ESP_OK;
}

// 保存图像到 SD 卡
static esp_err_t save_image_to_sdcard(const char *filename, const uint8_t *data, size_t len)
{
    FILE *file = fopen(filename, "wb");
    if (file == NULL) {
        ESP_LOGE(TAG, "无法创建文件 %s", filename);
        return ESP_FAIL;
    }
    
    size_t written = fwrite(data, 1, len, file);
    fclose(file);
    
    if (written != len) {
        ESP_LOGE(TAG, "写入文件失败 %s", filename);
        return ESP_FAIL;
    }
    
    ESP_LOGI(TAG, "图像已保存到 %s", filename);
    return ESP_OK;
}

void app_main(void)
{
    // 初始化 SD 卡
    if (init_sdcard() != ESP_OK) {
        ESP_LOGE(TAG, "SD 卡初始化失败");
        return;
    }
    
    // 初始化摄像头
    esp_err_t err = esp_camera_init(&camera_config);
    if (err != ESP_OK) {
        ESP_LOGE(TAG, "摄像头初始化失败 (0x%x)", err);
        return;
    }
    
    // 获取摄像头传感器对象
    sensor_t * sensor = esp_camera_sensor_get();
    if (sensor == NULL) {
        ESP_LOGE(TAG, "获取摄像头传感器失败");
        return;
    }
    
    // 设置摄像头参数
    sensor->set_brightness(sensor, 0);
    sensor->set_contrast(sensor, 0);
    sensor->set_saturation(sensor, 0);
    
    ESP_LOGI(TAG, "摄像头初始化成功");
    
    // 拍摄 5 张照片
    for (int i = 0; i < 5; i++) {
        // 获取一帧图像
        camera_fb_t *fb = esp_camera_fb_get();
        if (!fb) {
            ESP_LOGE(TAG, "获取摄像头帧失败");
            continue;
        }
        
        // 生成文件名
        char filename[32];
        snprintf(filename, sizeof(filename), "/sdcard/image_%d.jpg", i);
        
        // 保存图像到 SD 卡
        save_image_to_sdcard(filename, fb->buf, fb->len);
        
        // 释放帧缓冲区
        esp_camera_fb_return(fb);
        
        // 等待 1 秒
        vTaskDelay(pdMS_TO_TICKS(1000));
    }
    
    ESP_LOGI(TAG, "示例完成");
}
```



          
# ESP32-S3 驱动 OV2640 摄像头教程（续）

## 示例2：网页图传（续）

```c
void app_main(void)
{
    // 初始化 Wi-Fi
    init_wifi();
    
    // 初始化摄像头
    esp_err_t err = esp_camera_init(&camera_config);
    if (err != ESP_OK) {
        ESP_LOGE(TAG, "摄像头初始化失败 (0x%x)", err);
        return;
    }
    
    // 获取摄像头传感器对象
    sensor_t * sensor = esp_camera_sensor_get();
    if (sensor == NULL) {
        ESP_LOGE(TAG, "获取摄像头传感器失败");
        return;
    }
    
    // 设置摄像头参数
    sensor->set_brightness(sensor, 0);
    sensor->set_contrast(sensor, 0);
    sensor->set_saturation(sensor, 0);
    
    ESP_LOGI(TAG, "摄像头初始化成功");
    
    // 启动 HTTP 服务器
    httpd_handle_t server = start_webserver();
    if (server == NULL) {
        ESP_LOGE(TAG, "启动 HTTP 服务器失败");
        return;
    }
    
    // 获取 IP 地址
    tcpip_adapter_ip_info_t ip_info;
    ESP_ERROR_CHECK(tcpip_adapter_get_ip_info(TCPIP_ADAPTER_IF_STA, &ip_info));
    ESP_LOGI(TAG, "摄像头服务器已启动，请访问 http://"IPSTR":%d", IP2STR(&ip_info.ip), 80);
    
    // 主循环
    while (1) {
        vTaskDelay(1000 / portTICK_PERIOD_MS);
    }
}
```

## 常见问题与解决方案

### 1. 摄像头初始化失败

如果摄像头初始化失败，可能的原因和解决方法：

- **硬件连接问题**：
  - 检查所有引脚连接是否正确
  - 确保摄像头电源供应稳定（通常需要 3.3V）
  - 检查 SCCB（I2C）连接是否正确

- **引脚冲突**：
  - 确保所选引脚没有被其他功能占用
  - 检查是否有引脚被配置为其他功能

- **摄像头型号不匹配**：
  - 确认摄像头型号是否为 OV2640
  - 如果使用其他型号，需要修改相应的驱动代码

### 2. 图像质量问题

如果图像质量不佳，可能的原因和解决方法：

- **分辨率设置**：
  - 尝试不同的分辨率设置（FRAMESIZE_QVGA、FRAMESIZE_VGA 等）
  - 较低分辨率可能有更高的帧率和更稳定的传输

- **JPEG 质量**：
  - 调整 JPEG 压缩质量（0-63，数值越小质量越高）
  - 较低的质量可以提高传输速度，但会降低图像清晰度

- **光照条件**：
  - 确保摄像头有足够的光照
  - 调整亮度、对比度和饱和度参数

### 3. 内存不足

ESP32-S3 处理高分辨率图像可能会遇到内存不足的问题：

- **降低分辨率**：
  - 使用较低的分辨率（如 QVGA 320x240）
  - 减少帧缓冲区数量（fb_count）

- **优化内存分配**：
  - 增加 ESP32-S3 的 PSRAM 使用
  - 在 menuconfig 中启用 PSRAM 支持

- **减少其他任务**：
  - 减少同时运行的任务数量
  - 优化任务栈大小

### 4. Wi-Fi 连接问题

网页图传依赖于稳定的 Wi-Fi 连接：

- **Wi-Fi 信号强度**：
  - 确保 ESP32-S3 在 Wi-Fi 覆盖范围内
  - 使用信号强度更好的路由器或中继器

- **网络拥塞**：
  - 减少同一网络中的其他设备数量
  - 使用 5GHz 频段（如果支持）

- **固定 IP 地址**：
  - 为 ESP32-S3 设置静态 IP 地址，避免 DHCP 延迟

## 高级应用

### 1. 人脸检测

ESP32-S3 具有强大的 AI 处理能力，可以实现简单的人脸检测功能：

```c
#include "esp_face_detect.h"
#include "esp_face_recognition.h"

// 人脸检测任务
void face_detection_task(void *pvParameters)
{
    // 初始化人脸检测
    face_detection_init();
    
    while (1) {
        // 获取一帧图像
        camera_fb_t *fb = esp_camera_fb_get();
        if (!fb) {
            ESP_LOGE(TAG, "获取摄像头帧失败");
            vTaskDelay(pdMS_TO_TICKS(100));
            continue;
        }
        
        // 检测人脸
        box_array_t *boxes = face_detect(fb->buf, fb->len, fb->width, fb->height);
        
        if (boxes) {
            // 处理检测结果
            ESP_LOGI(TAG, "检测到 %d 个人脸", boxes->len);
            
            // 释放检测结果
            free(boxes);
        }
        
        // 释放帧缓冲区
        esp_camera_fb_return(fb);
        
        // 短暂延时
        vTaskDelay(pdMS_TO_TICKS(100));
    }
}
```

### 2. 图像处理

ESP32-S3 可以进行简单的图像处理操作：

```c
// 灰度转换函数
void convert_to_grayscale(uint8_t *rgb565_buffer, uint8_t *gray_buffer, int width, int height)
{
    for (int i = 0; i < width * height; i++) {
        uint16_t pixel = rgb565_buffer[i * 2] | (rgb565_buffer[i * 2 + 1] << 8);
        
        // 从 RGB565 提取 RGB 分量
        uint8_t r = (pixel >> 11) & 0x1F;
        uint8_t g = (pixel >> 5) & 0x3F;
        uint8_t b = pixel & 0x1F;
        
        // 转换为 8 位
        r = (r * 255) / 31;
        g = (g * 255) / 63;
        b = (b * 255) / 31;
        
        // 计算灰度值 (0.3R + 0.59G + 0.11B)
        gray_buffer[i] = (uint8_t)(0.3 * r + 0.59 * g + 0.11 * b);
    }
}
```

### 3. 视频录制

ESP32-S3 可以将摄像头捕获的图像保存为视频文件：

```c
#include "esp_vfs_fat.h"
#include "sdmmc_cmd.h"

// 视频录制任务
void video_recording_task(void *pvParameters)
{
    // 打开文件
    FILE *video_file = fopen("/sdcard/video.mjpeg", "wb");
    if (!video_file) {
        ESP_LOGE(TAG, "无法创建视频文件");
        return;
    }
    
    // 录制时间（10 秒）
    int recording_time_ms = 10000;
    int start_time = esp_timer_get_time() / 1000;
    
    ESP_LOGI(TAG, "开始录制视频...");
    
    while ((esp_timer_get_time() / 1000 - start_time) < recording_time_ms) {
        // 获取一帧图像
        camera_fb_t *fb = esp_camera_fb_get();
        if (!fb) {
            ESP_LOGE(TAG, "获取摄像头帧失败");
            vTaskDelay(pdMS_TO_TICKS(10));
            continue;
        }
        
        // 写入 MJPEG 帧标记
        fprintf(video_file, "--BOUNDARY\r\n");
        fprintf(video_file, "Content-Type: image/jpeg\r\n");
        fprintf(video_file, "Content-Length: %d\r\n\r\n", fb->len);
        
        // 写入图像数据
        fwrite(fb->buf, 1, fb->len, video_file);
        fprintf(video_file, "\r\n");
        
        // 释放帧缓冲区
        esp_camera_fb_return(fb);
        
        // 短暂延时（控制帧率）
        vTaskDelay(pdMS_TO_TICKS(50));  // 约 20 FPS
    }
    
    // 关闭文件
    fclose(video_file);
    
    ESP_LOGI(TAG, "视频录制完成");
}
```

## 总结

本教程详细介绍了如何在 ESP32-S3 Korvo2 v3 开发板上驱动 OV2640 摄像头，包括硬件连接、摄像头初始化、图像获取以及实用示例。通过 `esp_camera` 组件，ESP32-S3 可以轻松地与 OV2640 摄像头进行通信，获取高质量的图像数据。

ESP32-S3 强大的处理能力和丰富的外设接口使其成为图像处理和视觉应用的理想平台。无论是简单的图像捕获、网页图传，还是更复杂的人脸检测、图像处理和视频录制，ESP32-S3 都能胜任。

在实际应用中，应根据具体需求选择合适的分辨率、图像格式和处理方式，以平衡图像质量、处理速度和内存占用。同时，还应注意优化代码，合理利用 ESP32-S3 的资源，确保系统的稳定性和可靠性。

## 参考资料

- [ESP-IDF 编程指南](https://docs.espressif.com/projects/esp-idf/zh_CN/latest/esp32s3/index.html)
- [ESP32-S3 技术规格书](https://www.espressif.com/sites/default/files/documentation/esp32-s3_datasheet_cn.pdf)
- [ESP32-S3 Korvo2 v3 开发板用户指南](https://docs.espressif.com/projects/esp-adf/en/latest/design-guide/dev-boards/board-esp32-s3-korvo-2.html)
- [esp32-camera 组件文档](https://github.com/espressif/esp32-camera)
- [OV2640 数据手册](https://www.uctronics.com/download/cam_module/OV2640DS.pdf)
