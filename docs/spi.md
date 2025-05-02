


          
# ESP-IDF SPI 使用教程

## 目录
- [简介](#简介)
- [SPI 基础概念](#spi-基础概念)
- [ESP-IDF 中的 SPI 配置](#esp-idf-中的-spi-配置)
- [ST7789 LCD 显示屏介绍](#st7789-lcd-显示屏介绍)
- [驱动 ST7789 显示屏的完整示例](#驱动-st7789-显示屏的完整示例)
- [常见问题与解决方案](#常见问题与解决方案)
- [高级主题](#高级主题)
- [总结](#总结)
- [参考资料](#参考资料)

## 简介

SPI（Serial Peripheral Interface，串行外设接口）是一种同步串行通信接口，广泛应用于微控制器与各种外设（如显示屏、存储器、传感器等）之间的通信。ESP32 系列芯片提供了多个 SPI 控制器，可以同时与多个 SPI 设备通信。

本教程将详细介绍如何在 ESP-IDF 框架下使用 SPI 功能，并以驱动 ST7789 LCD 显示屏为例，展示 SPI 的初始化和数据读写操作。

## SPI 基础概念

### 1. SPI 通信原理

SPI 是一种全双工、同步的串行通信协议，通常包含以下四条信号线：

- **SCLK（Serial Clock）**：时钟信号，由主设备产生
- **MOSI（Master Out Slave In）**：主设备输出，从设备输入的数据线
- **MISO（Master In Slave Out）**：主设备输入，从设备输出的数据线
- **CS/SS（Chip Select/Slave Select）**：片选信号，用于选择要通信的从设备

### 2. SPI 工作模式

SPI 有四种工作模式（Mode 0-3），由时钟极性（CPOL）和时钟相位（CPHA）决定：

| 模式 | CPOL | CPHA | 时钟空闲状态 | 数据采样时刻 |
|------|------|------|------------|------------|
| 0    | 0    | 0    | 低电平      | 上升沿      |
| 1    | 0    | 1    | 低电平      | 下降沿      |
| 2    | 1    | 0    | 高电平      | 下降沿      |
| 3    | 1    | 1    | 高电平      | 上升沿      |

### 3. ESP32 的 SPI 控制器

ESP32 通常有以下几个 SPI 控制器：

- **SPI0/SPI1**：通常用于连接 Flash 和 PSRAM，不建议用户使用
- **SPI2/SPI3（HSPI/VSPI）**：可供用户使用的通用 SPI 控制器

## ESP-IDF 中的 SPI 配置

ESP-IDF 提供了两种使用 SPI 的方式：

1. **SPI 主机驱动（spi_master.h）**：适用于通用 SPI 主机通信
2. **SPI 总线驱动（spi_bus.h）**：更高级的 API，支持在同一总线上连接多个设备

本教程将使用 SPI 主机驱动来演示如何驱动 ST7789 显示屏。

### 1. 包含必要的头文件

```c
#include "driver/spi_master.h"
#include "driver/gpio.h"
#include "esp_log.h"
```

### 2. SPI 配置步骤

在 ESP-IDF 中配置 SPI 通常遵循以下步骤：

1. 初始化 SPI 总线
2. 添加 SPI 设备
3. 发送/接收数据
4. 释放资源（如需要）

## ST7789 LCD 显示屏介绍

ST7789 是一款常用的 LCD 控制器，通常用于驱动 240x280 分辨率的彩色 TFT 显示屏。它支持 SPI 接口，可以轻松与 ESP32 等微控制器连接。

### 1. ST7789 引脚定义

ST7789 显示屏通常需要以下几个引脚连接：

- **SCL/SCK**：SPI 时钟信号
- **SDA/MOSI**：SPI 数据输入
- **RES/RST**：复位信号
- **DC/RS**：数据/命令选择信号
- **CS**：片选信号
- **BLK**：背光控制（可选）

### 2. ST7789 通信协议

ST7789 通过 SPI 接口接收命令和数据。DC 引脚用于区分命令和数据：

- **DC = 低电平**：发送命令
- **DC = 高电平**：发送数据

## 驱动 ST7789 显示屏的完整示例

下面是一个完整的示例，展示如何使用 ESP-IDF 驱动 ST7789 显示屏：

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "driver/spi_master.h"
#include "driver/gpio.h"
#include "esp_log.h"

static const char *TAG = "st7789_example";

// ST7789 显示屏分辨率
#define ST7789_WIDTH  240
#define ST7789_HEIGHT 280

// 定义 GPIO 引脚
#define PIN_NUM_MISO -1  // 不使用 MISO
#define PIN_NUM_MOSI 19
#define PIN_NUM_CLK  18
#define PIN_NUM_CS   5
#define PIN_NUM_DC   16
#define PIN_NUM_RST  23
#define PIN_NUM_BLK  4

// ST7789 命令
#define ST7789_NOP     0x00
#define ST7789_SWRESET 0x01
#define ST7789_SLPIN   0x10
#define ST7789_SLPOUT  0x11
#define ST7789_NORON   0x13
#define ST7789_INVOFF  0x20
#define ST7789_INVON   0x21
#define ST7789_DISPOFF 0x28
#define ST7789_DISPON  0x29
#define ST7789_CASET   0x2A
#define ST7789_RASET   0x2B
#define ST7789_RAMWR   0x2C
#define ST7789_COLMOD  0x3A
#define ST7789_MADCTL  0x36

// 颜色定义 (RGB565 格式)
#define COLOR_BLACK   0x0000
#define COLOR_BLUE    0x001F
#define COLOR_RED     0xF800
#define COLOR_GREEN   0x07E0
#define COLOR_CYAN    0x07FF
#define COLOR_MAGENTA 0xF81F
#define COLOR_YELLOW  0xFFE0
#define COLOR_WHITE   0xFFFF

// SPI 句柄
static spi_device_handle_t spi_dev = NULL;

// 发送命令到 ST7789
static void st7789_cmd(spi_device_handle_t spi, const uint8_t cmd)
{
    esp_err_t ret;
    spi_transaction_t t;
    
    // 设置 DC 为低电平，表示发送命令
    gpio_set_level(PIN_NUM_DC, 0);
    
    // 发送命令
    memset(&t, 0, sizeof(t));
    t.length = 8;                   // 命令长度为 8 位
    t.tx_buffer = &cmd;             // 命令数据
    t.user = (void*)0;              // 用户数据，用于区分命令和数据
    
    ret = spi_device_polling_transmit(spi, &t);
    assert(ret == ESP_OK);
}

// 发送数据到 ST7789
static void st7789_data(spi_device_handle_t spi, const uint8_t *data, int len)
{
    esp_err_t ret;
    spi_transaction_t t;
    
    if (len == 0) return;
    
    // 设置 DC 为高电平，表示发送数据
    gpio_set_level(PIN_NUM_DC, 1);
    
    // 发送数据
    memset(&t, 0, sizeof(t));
    t.length = len * 8;             // 数据长度，单位为位
    t.tx_buffer = data;             // 数据缓冲区
    t.user = (void*)1;              // 用户数据，用于区分命令和数据
    
    ret = spi_device_polling_transmit(spi, &t);
    assert(ret == ESP_OK);
}

// 初始化 ST7789 显示屏
static void st7789_init(spi_device_handle_t spi)
{
    // 复位 ST7789
    gpio_set_level(PIN_NUM_RST, 0);
    vTaskDelay(pdMS_TO_TICKS(100));
    gpio_set_level(PIN_NUM_RST, 1);
    vTaskDelay(pdMS_TO_TICKS(100));
    
    // 打开背光
    gpio_set_level(PIN_NUM_BLK, 1);
    
    // 发送初始化命令序列
    st7789_cmd(spi, ST7789_SWRESET);    // 软件复位
    vTaskDelay(pdMS_TO_TICKS(150));
    
    st7789_cmd(spi, ST7789_SLPOUT);     // 退出睡眠模式
    vTaskDelay(pdMS_TO_TICKS(120));
    
    st7789_cmd(spi, ST7789_COLMOD);     // 设置颜色模式
    {
        uint8_t data = 0x55;            // 16 位/像素
        st7789_data(spi, &data, 1);
    }
    vTaskDelay(pdMS_TO_TICKS(10));
    
    st7789_cmd(spi, ST7789_MADCTL);     // 设置内存访问控制
    {
        uint8_t data = 0x00;            // 行地址顺序：从上到下，列地址顺序：从左到右
        st7789_data(spi, &data, 1);
    }
    
    st7789_cmd(spi, ST7789_INVON);      // 打开反显
    vTaskDelay(pdMS_TO_TICKS(10));
    
    st7789_cmd(spi, ST7789_NORON);      // 打开正常显示模式
    vTaskDelay(pdMS_TO_TICKS(10));
    
    st7789_cmd(spi, ST7789_DISPON);     // 打开显示
    vTaskDelay(pdMS_TO_TICKS(100));
    
    ESP_LOGI(TAG, "ST7789 初始化完成");
}

// 设置显示区域
static void st7789_set_window(spi_device_handle_t spi, uint16_t x0, uint16_t y0, uint16_t x1, uint16_t y1)
{
    // 设置列地址范围
    st7789_cmd(spi, ST7789_CASET);
    {
        uint8_t data[4];
        data[0] = (x0 >> 8) & 0xFF;
        data[1] = x0 & 0xFF;
        data[2] = (x1 >> 8) & 0xFF;
        data[3] = x1 & 0xFF;
        st7789_data(spi, data, 4);
    }
    
    // 设置行地址范围
    st7789_cmd(spi, ST7789_RASET);
    {
        uint8_t data[4];
        data[0] = (y0 >> 8) & 0xFF;
        data[1] = y0 & 0xFF;
        data[2] = (y1 >> 8) & 0xFF;
        data[3] = y1 & 0xFF;
        st7789_data(spi, data, 4);
    }
    
    // 准备写入显存
    st7789_cmd(spi, ST7789_RAMWR);
}

// 填充矩形区域
static void st7789_fill_rect(spi_device_handle_t spi, uint16_t x, uint16_t y, uint16_t w, uint16_t h, uint16_t color)
{
    uint32_t count = w * h;
    uint16_t buffer[32];  // 缓冲区，用于批量发送数据
    
    // 设置显示区域
    st7789_set_window(spi, x, y, x + w - 1, y + h - 1);
    
    // 填充颜色缓冲区
    for (int i = 0; i < 32; i++) {
        buffer[i] = color;
    }
    
    // 分批发送数据
    gpio_set_level(PIN_NUM_DC, 1);  // 数据模式
    
    while (count > 0) {
        uint32_t chunk_size = count > 32 ? 32 : count;
        
        // 发送一批数据
        spi_transaction_t t;
        memset(&t, 0, sizeof(t));
        t.length = chunk_size * 16;  // 16 位/像素
        t.tx_buffer = buffer;
        spi_device_polling_transmit(spi, &t);
        
        count -= chunk_size;
    }
}

// 绘制彩色条纹
static void st7789_draw_color_bars(spi_device_handle_t spi)
{
    uint16_t colors[] = {
        COLOR_RED, COLOR_GREEN, COLOR_BLUE, COLOR_CYAN,
        COLOR_MAGENTA, COLOR_YELLOW, COLOR_WHITE, COLOR_BLACK
    };
    
    int bar_width = ST7789_WIDTH / 8;
    
    for (int i = 0; i < 8; i++) {
        st7789_fill_rect(spi, i * bar_width, 0, bar_width, ST7789_HEIGHT, colors[i]);
    }
}

void app_main(void)
{
    ESP_LOGI(TAG, "初始化 SPI 总线");
    
    // 初始化非 SPI 的 GPIO 引脚
    gpio_set_direction(PIN_NUM_DC, GPIO_MODE_OUTPUT);
    gpio_set_direction(PIN_NUM_RST, GPIO_MODE_OUTPUT);
    gpio_set_direction(PIN_NUM_BLK, GPIO_MODE_OUTPUT);
    
    // 配置 SPI 总线
    spi_bus_config_t buscfg = {
        .miso_io_num = PIN_NUM_MISO,
        .mosi_io_num = PIN_NUM_MOSI,
        .sclk_io_num = PIN_NUM_CLK,
        .quadwp_io_num = -1,
        .quadhd_io_num = -1,
        .max_transfer_sz = ST7789_WIDTH * ST7789_HEIGHT * 2 + 8
    };
    
    // 初始化 SPI 总线
    ESP_ERROR_CHECK(spi_bus_initialize(SPI2_HOST, &buscfg, SPI_DMA_CH_AUTO));
    
    // 配置 SPI 设备
    spi_device_interface_config_t devcfg = {
        .clock_speed_hz = 40 * 1000 * 1000,  // 40 MHz
        .mode = 0,                           // SPI 模式 0
        .spics_io_num = PIN_NUM_CS,          // CS 引脚
        .queue_size = 7,                     // 事务队列大小
        .pre_cb = NULL,                      // 传输前回调
        .post_cb = NULL,                     // 传输后回调
    };
    
    // 添加 SPI 设备
    ESP_ERROR_CHECK(spi_bus_add_device(SPI2_HOST, &devcfg, &spi_dev));
    
    // 初始化 ST7789 显示屏
    st7789_init(spi_dev);
    
    // 清屏（黑色）
    st7789_fill_rect(spi_dev, 0, 0, ST7789_WIDTH, ST7789_HEIGHT, COLOR_BLACK);
    vTaskDelay(pdMS_TO_TICKS(1000));
    
    // 绘制彩色条纹
    ESP_LOGI(TAG, "绘制彩色条纹");
    st7789_draw_color_bars(spi_dev);
    
    // 循环显示不同颜色
    int color_index = 0;
    uint16_t colors[] = {
        COLOR_RED, COLOR_GREEN, COLOR_BLUE, COLOR_CYAN,
        COLOR_MAGENTA, COLOR_YELLOW, COLOR_WHITE, COLOR_BLACK
    };
    
    while (1) {
        ESP_LOGI(TAG, "填充颜色: %d", color_index);
        st7789_fill_rect(spi_dev, 0, 0, ST7789_WIDTH, ST7789_HEIGHT, colors[color_index]);
        
        color_index = (color_index + 1) % 8;
        vTaskDelay(pdMS_TO_TICKS(1000));
    }
    
    // 释放资源（实际上不会执行到这里）
    ESP_ERROR_CHECK(spi_bus_remove_device(spi_dev));
    ESP_ERROR_CHECK(spi_bus_free(SPI2_HOST));
}
```

## 常见问题与解决方案

### 1. SPI 通信失败

如果 SPI 通信失败，可能的原因和解决方法：

- **硬件连接问题**：
  - 检查 MOSI、SCLK、CS、DC、RST 等引脚是否正确连接
  - 确保 SPI 设备的电源供应正常

- **SPI 配置不正确**：
  - 检查 SPI 模式（CPOL 和 CPHA）是否与设备要求一致
  - 确认 SPI 时钟频率是否在设备支持范围内

- **引脚冲突**：
  - 确保所选引脚没有被其他功能占用
  - 检查是否有引脚被配置为其他功能（如 GPIO 输出）

### 2. 显示异常

如果 ST7789 显示屏显示异常，可能的原因和解决方法：

- **初始化命令不正确**：
  - 检查初始化命令序列是否符合 ST7789 数据手册要求
  - 确认命令参数是否正确

- **显示方向问题**：
  - 调整 MADCTL 命令参数，改变显示方向

- **颜色格式不正确**：
  - 确认 COLMOD 命令设置的颜色格式与发送的数据格式一致

### 3. 提高 SPI 传输速度

对于显示屏等需要大量数据传输的应用，可以采取以下措施提高 SPI 传输速度：

- 使用 DMA 传输
- 增加 SPI 时钟频率
- 批量发送数据，减少传输次数
- 使用事务队列，而不是轮询传输

## 高级主题

### 1. 使用 DMA 传输

ESP-IDF 的 SPI 驱动支持 DMA 传输，可以显著提高传输效率。在初始化 SPI 总线时，通过设置 `SPI_DMA_CH_AUTO` 参数启用 DMA：

```c
ESP_ERROR_CHECK(spi_bus_initialize(SPI2_HOST, &buscfg, SPI_DMA_CH_AUTO));
```

### 2. 使用事务队列

除了轮询传输外，ESP-IDF 还支持使用事务队列进行 SPI 传输，可以提高系统响应性：

```c
// 提交事务到队列
spi_device_queue_trans(spi_dev, &t, portMAX_DELAY);

// 等待事务完成
spi_device_get_trans_result(spi_dev, &ret_trans, portMAX_DELAY);
```

### 3. 传输前后回调

ESP-IDF 支持在 SPI 传输前后执行回调函数，可以用于自动控制 DC 引脚等：

```c
// DC 引脚控制回调
void lcd_spi_pre_transfer_callback(spi_transaction_t *t)
{
    int dc = (int)t->user;
    gpio_set_level(PIN_NUM_DC, dc);
}

// 配置 SPI 设备时设置回调
devcfg.pre_cb = lcd_spi_pre_transfer_callback;
```

## 总结

本教程详细介绍了 ESP-IDF 中 SPI 的使用方法，包括配置、基本操作以及驱动 ST7789 LCD 显示屏的完整示例。通过掌握这些知识，您可以轻松地在 ESP32 项目中集成各种 SPI 设备。

SPI 是一种高速、可靠的通信协议，适用于连接各种外设，如显示屏、存储器、传感器等。ESP32 的 SPI 控制器提供了丰富的功能和灵活的配置选项，可以满足大多数应用场景的需求。

在后续的教程中，我们将介绍更多 ESP-IDF 的功能，如 UART 等通信协议的使用方法。

## 参考资料

- [ESP-IDF 编程指南 - SPI 主机驱动程序](https://docs.espressif.com/projects/esp-idf/zh_CN/latest/esp32/api-reference/peripherals/spi_master.html)
- [ST7789 数据手册](https://www.newhavendisplay.com/appnotes/datasheets/LCDs/ST7789V.pdf)
- [ESP32 技术规格书](https://www.espressif.com/sites/default/files/documentation/esp32_datasheet_cn.pdf)

