


          
# ESP-IDF I2C 使用教程

## 目录
- [简介](#简介)
- [I2C 基础概念](#i2c-基础概念)
- [ESP-IDF 中的 I2C 配置](#esp-idf-中的-i2c-配置)
- [I2C 主机总线驱动](#i2c-主机总线驱动)
- [I2C 基本操作](#i2c-基本操作)
- [实用示例](#实用示例)
- [常见问题与解决方案](#常见问题与解决方案)

## 简介

I2C（Inter-Integrated Circuit，内部集成电路总线）是一种串行通信总线，由飞利浦公司开发，用于连接微控制器和各种外围设备，如传感器、EEPROM、LCD驱动器等。ESP32 系列芯片支持标准模式（100 kbit/s）、快速模式（400 kbit/s）以及高速模式（最高可达 3.4 Mbit/s）的 I2C 通信。

ESP-IDF 提供了完整的 I2C 驱动程序，支持 I2C 主机和从机模式。本教程将重点介绍 ESP-IDF 最新的 I2C 主机总线（I2C Master Bus）API 的使用方法。

## I2C 基础概念

### 1. I2C 总线特点

- **双线制**：只需要两根线（SDA 和 SCL）即可实现通信
- **主从架构**：一个主设备可以控制多个从设备
- **地址寻址**：每个从设备都有唯一的地址
- **双向通信**：支持主设备向从设备写入数据，也支持从从设备读取数据

### 2. I2C 信号线

- **SDA（串行数据线）**：用于传输数据
- **SCL（串行时钟线）**：由主设备提供的时钟信号

### 3. I2C 通信协议

I2C 通信过程包括以下几个关键步骤：
- **起始条件**：SCL 为高电平时，SDA 从高电平切换到低电平
- **地址传输**：主设备发送 7 位或 10 位从设备地址和读/写位
- **数据传输**：按字节传输数据，每个字节后跟一个应答位
- **停止条件**：SCL 为高电平时，SDA 从低电平切换到高电平

## ESP-IDF 中的 I2C 配置

ESP-IDF 提供了两种 I2C 驱动 API：
1. **传统 I2C 驱动 API**：较早的 API，提供基本的 I2C 功能
2. **I2C 主机总线驱动 API**：较新的 API，提供更灵活、更高级的功能

本教程将重点介绍 **I2C 主机总线驱动 API**，这是 ESP-IDF 最新推荐的 I2C 使用方式。

### 1. 包含必要的头文件

```c
#include "driver/i2c_master.h"
```

### 2. I2C 主机总线驱动配置步骤

使用 I2C 主机总线驱动 API 的基本步骤如下：

1. 创建 I2C 主机总线配置结构体
2. 安装 I2C 主机总线驱动
3. 创建 I2C 设备配置结构体
4. 分配 I2C 设备句柄
5. 使用 I2C 设备句柄进行读写操作
6. 释放资源

## I2C 主机总线驱动

### 1. 创建和配置 I2C 主机总线

```c
#include <stdio.h>
#include "driver/i2c_master.h"

#define I2C_MASTER_SCL_IO           22      // GPIO 用于 SCL 信号
#define I2C_MASTER_SDA_IO           21      // GPIO 用于 SDA 信号
#define I2C_MASTER_FREQ_HZ          400000  // I2C 主机时钟频率
#define I2C_MASTER_TIMEOUT_MS       1000    // I2C 通信超时时间

void app_main(void)
{
    // 创建 I2C 主机总线配置结构体
    i2c_master_bus_config_t i2c_mst_config = {
        .clk_source = I2C_CLK_SRC_DEFAULT,  // 时钟源
        .i2c_port = I2C_NUM_0,              // I2C 端口号
        .scl_io_num = I2C_MASTER_SCL_IO,    // SCL GPIO 号
        .sda_io_num = I2C_MASTER_SDA_IO,    // SDA GPIO 号
        .glitch_ignore_cnt = 7,             // 忽略毛刺计数
        .flags.enable_internal_pullup = true, // 启用内部上拉电阻
    };
    
    // 安装 I2C 主机总线驱动
    i2c_master_bus_handle_t bus_handle;
    ESP_ERROR_CHECK(i2c_new_master_bus(&i2c_mst_config, &bus_handle));
    
    // 设置 I2C 总线时钟频率
    ESP_ERROR_CHECK(i2c_master_bus_set_clock_source(bus_handle, I2C_CLK_SRC_DEFAULT));
    ESP_ERROR_CHECK(i2c_master_bus_set_freq(bus_handle, I2C_MASTER_FREQ_HZ));
    
    printf("I2C 主机总线初始化成功\n");
    
    // 在这里添加 I2C 设备配置和操作
    
    // 释放 I2C 主机总线
    // ESP_ERROR_CHECK(i2c_del_master_bus(bus_handle));
}
```

### 2. 配置 I2C 设备

一旦创建了 I2C 主机总线，就可以在该总线上配置多个 I2C 设备：

```c
// 假设有一个地址为 0x48 的 I2C 设备
#define I2C_DEVICE_ADDR             0x48

// 创建 I2C 设备配置结构体
i2c_device_config_t dev_cfg = {
    .dev_addr_length = I2C_ADDR_BIT_LEN_7,    // 7 位地址长度
    .device_address = I2C_DEVICE_ADDR,        // 设备地址
    .scl_speed_hz = I2C_MASTER_FREQ_HZ,       // SCL 频率
};

// 分配 I2C 设备句柄
i2c_master_dev_handle_t dev_handle;
ESP_ERROR_CHECK(i2c_master_bus_add_device(bus_handle, &dev_cfg, &dev_handle));

printf("I2C 设备 (地址: 0x%02X) 添加成功\n", I2C_DEVICE_ADDR);
```

## I2C 基本操作

### 1. 写入数据

使用 I2C 主机总线驱动 API 写入数据的方法：

```c
// 准备要写入的数据
uint8_t write_buffer[2] = {0x01, 0x02};  // 示例数据

// 写入数据
esp_err_t err = i2c_master_transmit(dev_handle, write_buffer, sizeof(write_buffer), I2C_MASTER_TIMEOUT_MS);
if (err == ESP_OK) {
    printf("数据写入成功\n");
} else {
    printf("数据写入失败: %s\n", esp_err_to_name(err));
}
```

### 2. 读取数据

使用 I2C 主机总线驱动 API 读取数据的方法：

```c
// 准备接收数据的缓冲区
uint8_t read_buffer[2] = {0};

// 读取数据
esp_err_t err = i2c_master_receive(dev_handle, read_buffer, sizeof(read_buffer), I2C_MASTER_TIMEOUT_MS);
if (err == ESP_OK) {
    printf("数据读取成功: 0x%02X 0x%02X\n", read_buffer[0], read_buffer[1]);
} else {
    printf("数据读取失败: %s\n", esp_err_to_name(err));
}
```

### 3. 写后读操作

许多 I2C 设备需要先写入寄存器地址，然后再读取数据。I2C 主机总线驱动 API 提供了专门的函数来处理这种情况：

```c
// 准备写入的寄存器地址
uint8_t reg_addr = 0x00;  // 示例寄存器地址

// 准备接收数据的缓冲区
uint8_t read_buffer[2] = {0};

// 写后读操作
esp_err_t err = i2c_master_transmit_receive(dev_handle, &reg_addr, 1, read_buffer, sizeof(read_buffer), I2C_MASTER_TIMEOUT_MS);
if (err == ESP_OK) {
    printf("写后读操作成功: 0x%02X 0x%02X\n", read_buffer[0], read_buffer[1]);
} else {
    printf("写后读操作失败: %s\n", esp_err_to_name(err));
}
```

### 4. 队列传输

对于更复杂的 I2C 传输序列，可以使用队列传输功能：

```c
// 创建 I2C 传输描述符
i2c_master_transfer_t transfer = {
    .flags = 0,                  // 标志位
};

// 写入操作
uint8_t write_buffer[2] = {0x01, 0x02};
transfer.tx_data = write_buffer;
transfer.tx_size = sizeof(write_buffer);
transfer.rx_data = NULL;
transfer.rx_size = 0;

// 执行传输
esp_err_t err = i2c_master_transfer(dev_handle, &transfer, I2C_MASTER_TIMEOUT_MS);
if (err == ESP_OK) {
    printf("队列传输成功\n");
} else {
    printf("队列传输失败: %s\n", esp_err_to_name(err));
}
```

## 实用示例

### 示例1：读取 MPU6050 传感器数据

MPU6050 是一个常用的 6 轴运动传感器，通过 I2C 接口通信。以下是使用 I2C 主机总线驱动 API 读取 MPU6050 数据的示例：

```c
#include <stdio.h>
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "driver/i2c_master.h"
#include "esp_log.h"

static const char *TAG = "mpu6050_example";

#define I2C_MASTER_SCL_IO           22
#define I2C_MASTER_SDA_IO           21
#define I2C_MASTER_FREQ_HZ          400000
#define I2C_MASTER_TIMEOUT_MS       1000

#define MPU6050_ADDR                0x68    // MPU6050 设备地址
#define MPU6050_WHO_AM_I            0x75    // WHO_AM_I 寄存器地址
#define MPU6050_PWR_MGMT_1          0x6B    // 电源管理寄存器地址
#define MPU6050_ACCEL_XOUT_H        0x3B    // 加速度计数据起始寄存器地址

// 初始化 MPU6050
static esp_err_t mpu6050_init(i2c_master_dev_handle_t dev_handle)
{
    esp_err_t ret;
    uint8_t cmd[2];
    uint8_t data;
    
    // 检查设备 ID
    cmd[0] = MPU6050_WHO_AM_I;
    ret = i2c_master_transmit_receive(dev_handle, cmd, 1, &data, 1, I2C_MASTER_TIMEOUT_MS);
    if (ret != ESP_OK) {
        ESP_LOGE(TAG, "无法读取 WHO_AM_I 寄存器");
        return ret;
    }
    
    if (data != 0x68) {
        ESP_LOGE(TAG, "未找到 MPU6050 设备，WHO_AM_I = 0x%02X", data);
        return ESP_ERR_NOT_FOUND;
    }
    
    ESP_LOGI(TAG, "MPU6050 设备找到，WHO_AM_I = 0x%02X", data);
    
    // 唤醒 MPU6050
    cmd[0] = MPU6050_PWR_MGMT_1;
    cmd[1] = 0x00;  // 清除睡眠位
    ret = i2c_master_transmit(dev_handle, cmd, 2, I2C_MASTER_TIMEOUT_MS);
    if (ret != ESP_OK) {
        ESP_LOGE(TAG, "无法唤醒 MPU6050");
        return ret;
    }
    
    ESP_LOGI(TAG, "MPU6050 初始化成功");
    return ESP_OK;
}

// 读取 MPU6050 加速度计和陀螺仪数据
static esp_err_t mpu6050_read_sensor_data(i2c_master_dev_handle_t dev_handle)
{
    esp_err_t ret;
    uint8_t cmd = MPU6050_ACCEL_XOUT_H;
    uint8_t data[14];  // 加速度计和陀螺仪数据共 14 字节
    
    // 读取传感器数据
    ret = i2c_master_transmit_receive(dev_handle, &cmd, 1, data, 14, I2C_MASTER_TIMEOUT_MS);
    if (ret != ESP_OK) {
        ESP_LOGE(TAG, "无法读取传感器数据");
        return ret;
    }
    
    // 解析加速度计数据
    int16_t accel_x = (data[0] << 8) | data[1];
    int16_t accel_y = (data[2] << 8) | data[3];
    int16_t accel_z = (data[4] << 8) | data[5];
    
    // 解析陀螺仪数据
    int16_t gyro_x = (data[8] << 8) | data[9];
    int16_t gyro_y = (data[10] << 8) | data[11];
    int16_t gyro_z = (data[12] << 8) | data[13];
    
    // 打印传感器数据
    ESP_LOGI(TAG, "加速度计: X=%d, Y=%d, Z=%d", accel_x, accel_y, accel_z);
    ESP_LOGI(TAG, "陀螺仪: X=%d, Y=%d, Z=%d", gyro_x, gyro_y, gyro_z);
    
    return ESP_OK;
}

void app_main(void)
{
    // 创建 I2C 主机总线配置结构体
    i2c_master_bus_config_t i2c_mst_config = {
        .clk_source = I2C_CLK_SRC_DEFAULT,
        .i2c_port = I2C_NUM_0,
        .scl_io_num = I2C_MASTER_SCL_IO,
        .sda_io_num = I2C_MASTER_SDA_IO,
        .glitch_ignore_cnt = 7,
        .flags.enable_internal_pullup = true,
    };
    
    // 安装 I2C 主机总线驱动
    i2c_master_bus_handle_t bus_handle;
    ESP_ERROR_CHECK(i2c_new_master_bus(&i2c_mst_config, &bus_handle));
    
    // 设置 I2C 总线时钟频率
    ESP_ERROR_CHECK(i2c_master_bus_set_freq(bus_handle, I2C_MASTER_FREQ_HZ));
    
    // 创建 MPU6050 设备配置结构体
    i2c_device_config_t dev_cfg = {
        .dev_addr_length = I2C_ADDR_BIT_LEN_7,
        .device_address = MPU6050_ADDR,
        .scl_speed_hz = I2C_MASTER_FREQ_HZ,
    };
    
    // 分配 MPU6050 设备句柄
    i2c_master_dev_handle_t dev_handle;
    ESP_ERROR_CHECK(i2c_master_bus_add_device(bus_handle, &dev_cfg, &dev_handle));
    
    // 初始化 MPU6050
    ESP_ERROR_CHECK(mpu6050_init(dev_handle));
    
    // 循环读取传感器数据
    while (1) {
        mpu6050_read_sensor_data(dev_handle);
        vTaskDelay(pdMS_TO_TICKS(1000));  // 每秒读取一次
    }
    
    // 释放资源（实际上不会执行到这里）
    ESP_ERROR_CHECK(i2c_master_bus_rm_device(dev_handle));
    ESP_ERROR_CHECK(i2c_del_master_bus(bus_handle));
}
```

### 示例2：OLED 显示屏控制

SSD1306 是一种常用的 OLED 显示屏控制器，通过 I2C 接口通信。以下是使用 I2C 主机总线驱动 API 控制 SSD1306 OLED 显示屏的简化示例：

```c
#include <stdio.h>
#include <string.h>
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "driver/i2c_master.h"
#include "esp_log.h"

static const char *TAG = "ssd1306_example";

#define I2C_MASTER_SCL_IO           22
#define I2C_MASTER_SDA_IO           21
#define I2C_MASTER_FREQ_HZ          400000
#define I2C_MASTER_TIMEOUT_MS       1000

#define SSD1306_ADDR                0x3C    // SSD1306 设备地址
#define SSD1306_CONTROL_BYTE_CMD    0x00    // 控制字节: 命令
#define SSD1306_CONTROL_BYTE_DATA   0x40    // 控制字节: 数据

#define SSD1306_WIDTH               128
#define SSD1306_HEIGHT              64

// SSD1306 初始化命令
static const uint8_t ssd1306_init_cmd[] = {
    0xAE,         // 关闭显示
    0xD5, 0x80,   // 设置显示时钟分频比/振荡器频率
    0xA8, 0x3F,   // 设置多路复用比
    0xD3, 0x00,   // 设置显示偏移
    0x40,         // 设置显示开始行
    0x8D, 0x14,   // 电荷泵设置
    0x20, 0x00,   // 设置内存寻址模式
    0xA1,         // 段重映射
    0xC8,         // COM 输出扫描方向
    0xDA, 0x12,   // 设置 COM 引脚硬件配置
    0x81, 0xCF,   // 设置对比度控制
    0xD9, 0xF1,   // 设置预充电周期
    0xDB, 0x40,   // 设置 VCOMH 解除选择电平
    0xA4,         // 整个显示打开
    0xA6,         // 设置正常显示
    0xAF          // 开启显示
};

// 发送命令到 SSD1306
static esp_err_t ssd1306_send_cmd(i2c_master_dev_handle_t dev_handle, uint8_t cmd)
{
    uint8_t buf[2] = {SSD1306_CONTROL_BYTE_CMD, cmd};
    return i2c_master_transmit(dev_handle, buf, 2, I2C_MASTER_TIMEOUT_MS);
}

// 发送数据到 SSD1306
static esp_err_t ssd1306_send_data(i2c_master_dev_handle_t dev_handle, const uint8_t *data, size_t size)
{
    uint8_t *buf = malloc(size + 1);
    if (buf == NULL) {
        return ESP_ERR_NO_MEM;
    }
    
    buf[0] = SSD1306_CONTROL_BYTE_DATA;
    memcpy(buf + 1, data, size);
    
    esp_err_t ret = i2c_master_transmit(dev_handle, buf, size + 1, I2C_MASTER_TIMEOUT_MS);
    free(buf);
    
    return ret;
}

// 初始化 SSD1306
static esp_err_t ssd1306_init(i2c_master_dev_handle_t dev_handle)
{
    esp_err_t ret;
    
    // 发送初始化命令
    for (int i = 0; i < sizeof(ssd1306_init_cmd); i++) {
        ret = ssd1306_send_cmd(dev_handle, ssd1306_init_cmd[i]);
        if (ret != ESP_OK) {
            ESP_LOGE(TAG, "SSD1306 初始化失败，命令 0x%02X", ssd1306_init_cmd[i]);
            return ret;
        }
    }
    
    ESP_LOGI(TAG, "SSD1306 初始化成功");
    return ESP_OK;
}

// 清除 SSD1306 显示
static esp_err_t ssd1306_clear(i2c_master_dev_handle_t dev_handle)
{
    esp_err_t ret;
    
    // 设置列地址范围
    ret = ssd1306_send_cmd(dev_handle, 0x21);  // 设置列地址命令
    if (ret != ESP_OK) return ret;
    ret = ssd1306_send_cmd(dev_handle, 0);     // 列起始地址
    if (ret != ESP_OK) return ret;
    ret = ssd1306_send_cmd(dev_handle, SSD1306_WIDTH - 1);  // 列结束地址
    if (ret != ESP_OK) return ret;
    
    // 设置页地址范围
    ret = ssd1306_send_cmd(dev_handle, 0x22);  // 设置页地址命令
    if (ret != ESP_OK) return ret;
    ret = ssd1306_send_cmd(dev_handle, 0);     // 页起始地址
    if (ret != ESP_OK) return ret;
    ret = ssd1306_send_cmd(dev_handle, 7);     // 页结束地址
    if (ret != ESP_OK) return ret;
    
    // 清除显示缓冲区
    uint8_t zero[16] = {0};
    for (int i = 0; i < SSD1306_WIDTH * SSD1306_HEIGHT / 8 / 16; i++) {
        ret = ssd1306_send_data(dev_handle, zero, 16);
        if (ret != ESP_OK) return ret;
    }
    
    return ESP_OK;
}

// 显示简单图案
static esp_err_t ssd1306_display_pattern(i2c_master_dev_handle_t dev_handle)
{
    esp_err_t ret;
    
    // 设置列地址范围
    ret = ssd1306_send_cmd(dev_handle, 0x21);  // 设置列地址命令
    if (ret != ESP_OK) return ret;
    ret = ssd1306_send_cmd(dev_handle, 0);     // 列起始地址
    if (ret != ESP_OK) return ret;
    ret = ssd1306_send_cmd(dev_handle, SSD1306_WIDTH - 1);  // 列结束地址
    if (ret != ESP_OK) return ret;
    
    // 设置页地址范围
    ret = ssd1306_send_cmd(dev_handle, 0x22);  // 设置页地址命令
    if (ret != ESP_OK) return ret;
    ret = ssd1306_send_cmd(dev_handle, 0);     // 页起始地址
    if (ret != ESP_OK) return ret;
    ret = ssd1306_send_cmd(dev_handle, 7);     // 页结束地址
    if (ret != ESP_OK) return ret;
    
    // 创建棋盘图案
    uint8_t pattern[16];
    for (int i = 0; i < 16; i++) {
        pattern[i] = (i % 2) ? 0xAA : 0x55;
    }
    
    // 发送图案数据
    for (int i = 0; i < SSD1306_WIDTH * SSD1306_HEIGHT / 8 / 16; i++) {
        ret = ssd1306_send_data(dev_handle, pattern, 16);
        if (ret != ESP_OK) return ret;
    }
    
    return ESP_OK;
}

void app_main(void)
{
    // 创建 I2C 主机总线配置结构体
    i2c_master_bus_config_t i2c_mst_config = {
        .clk_source = I2C_CLK_SRC_DEFAULT,
        .i2c_port = I2C_NUM_0,
        .scl_io_num = I2C_MASTER_SCL_IO,
        .sda_io_num = I2C_MASTER_SDA_IO,
        .glitch_ignore_cnt = 7,
        .flags.enable_internal_pullup = true,
    };
    
    // 安装 I2C 主机总线驱动
    i2c_master_bus_handle_t bus_handle;
    ESP_ERROR_CHECK(i2c_new_master_bus(&i2c_mst_config, &bus_handle));
    
    // 设置 I2C 总线时钟频率
    ESP_ERROR_CHECK(i2c_master_bus_set_freq(bus_handle, I2C_MASTER_FREQ_HZ));
    
    // 创建 SSD1306 设备配置结构体
    i2c_device_config_t dev_cfg = {
        .dev_addr_length = I2C_ADDR_BIT_LEN_7,
        .device_address = SSD1306_ADDR,
        .scl_speed_hz = I2C_MASTER_FREQ_HZ,
    };
    
    // 分配 SSD1306 设备句柄
    i2c_master_dev_handle_t dev_handle;
    ESP_ERROR_CHECK(i2c_master_bus_add_device(bus_handle, &dev_cfg, &dev_handle));
    
    // 初始化 SSD1306
    ESP_ERROR_CHECK(ssd1306_init(dev_handle));
    
    // 清除显示
    ESP_ERROR_CHECK(ssd1306_clear(dev_handle));
    
    ESP_LOGI(TAG, "显示棋盘图案");
    ESP_ERROR_CHECK(ssd1306_display_pattern(dev_handle));
    
    // 循环显示不同图案
    int count = 0;
    while (1) {
        if (count % 2 == 0) {
            ESP_LOGI(TAG, "清除显示");
            ESP_ERROR_CHECK(ssd1306_clear(dev_handle));
        } else {
            ESP_LOGI(TAG, "显示棋盘图案");
            ESP_ERROR_CHECK(ssd1306_display_pattern(dev_handle));
        }
        
        count++;
        vTaskDelay(pdMS_TO_TICKS(2000));  // 每 2 秒切换一次
    }
    
    // 释放资源（实际上不会执行到这里）
    ESP_ERROR_CHECK(i2c_master_bus_rm_device(dev_handle));
    ESP_ERROR_CHECK(i2c_del_master_bus(bus_handle));
}
```




          
## 常见问题与解决方案（续）

### 1. I2C 通信失败（续）

- **时钟频率过高**：
  - 降低 I2C 时钟频率（例如从 400kHz 降至 100kHz）
  - 某些设备可能不支持高速 I2C 通信

- **总线冲突**：
  - 确保总线上没有多个主设备同时通信
  - 检查是否有其他设备占用了 I2C 总线

- **超时设置**：
  - 增加超时时间，特别是对于响应较慢的设备

### 2. I2C 设备地址冲突

当多个 I2C 设备具有相同地址时，会发生地址冲突。解决方法：

- 使用具有可配置地址的设备
- 使用 I2C 多路复用器将设备分配到不同的总线
- 使用多个 I2C 控制器（ESP32 有两个 I2C 控制器）

### 3. I2C 扫描工具

以下是一个简单的 I2C 扫描工具，用于检测总线上的设备：

```c
#include <stdio.h>
#include "driver/i2c_master.h"
#include "esp_log.h"

static const char *TAG = "i2c_scanner";

#define I2C_MASTER_SCL_IO           22
#define I2C_MASTER_SDA_IO           21
#define I2C_MASTER_FREQ_HZ          100000  // 使用较低频率进行扫描
#define I2C_MASTER_TIMEOUT_MS       10      // 短超时时间用于扫描

void app_main(void)
{
    // 创建 I2C 主机总线配置结构体
    i2c_master_bus_config_t i2c_mst_config = {
        .clk_source = I2C_CLK_SRC_DEFAULT,
        .i2c_port = I2C_NUM_0,
        .scl_io_num = I2C_MASTER_SCL_IO,
        .sda_io_num = I2C_MASTER_SDA_IO,
        .glitch_ignore_cnt = 7,
        .flags.enable_internal_pullup = true,
    };
    
    // 安装 I2C 主机总线驱动
    i2c_master_bus_handle_t bus_handle;
    ESP_ERROR_CHECK(i2c_new_master_bus(&i2c_mst_config, &bus_handle));
    
    // 设置 I2C 总线时钟频率
    ESP_ERROR_CHECK(i2c_master_bus_set_freq(bus_handle, I2C_MASTER_FREQ_HZ));
    
    printf("扫描 I2C 总线上的设备...\n");
    
    uint8_t count = 0;
    for (uint8_t i = 1; i < 128; i++) {  // 跳过地址 0（通用调用地址）
        i2c_device_config_t dev_cfg = {
            .dev_addr_length = I2C_ADDR_BIT_LEN_7,
            .device_address = i,
            .scl_speed_hz = I2C_MASTER_FREQ_HZ,
        };
        
        i2c_master_dev_handle_t dev_handle;
        esp_err_t ret = i2c_master_bus_add_device(bus_handle, &dev_cfg, &dev_handle);
        
        if (ret != ESP_OK) {
            continue;  // 跳过错误
        }
        
        // 尝试与设备通信
        uint8_t dummy;
        ret = i2c_master_receive(dev_handle, &dummy, 1, I2C_MASTER_TIMEOUT_MS);
        
        // 移除设备
        i2c_master_bus_rm_device(dev_handle);
        
        if (ret == ESP_OK || ret == ESP_ERR_INVALID_SIZE) {
            printf("找到设备，地址: 0x%02X\n", i);
            count++;
        }
    }
    
    printf("扫描完成，找到 %d 个设备\n", count);
    
    // 释放 I2C 主机总线
    ESP_ERROR_CHECK(i2c_del_master_bus(bus_handle));
}
```

### 4. 长线缆问题

当 I2C 总线线缆较长时，可能会出现信号质量问题。解决方法：

- 降低 I2C 时钟频率
- 使用更强的上拉电阻（例如 2.2kΩ 而不是 4.7kΩ）
- 考虑使用 I2C 总线缓冲器/中继器
- 确保线缆质量良好，最好使用屏蔽线

### 5. 多线程安全

ESP-IDF 的 I2C 主机总线驱动 API 是线程安全的，但在多任务环境中使用时，仍需注意以下几点：

- 避免多个任务同时访问同一个 I2C 设备
- 如果需要多任务访问，考虑使用互斥锁保护 I2C 操作
- 使用队列在任务之间传递 I2C 数据

## 高级主题

### 1. I2C 从机模式

除了主机模式外，ESP32 还支持 I2C 从机模式。但请注意，I2C 主机总线驱动 API 仅支持主机模式。如果需要使用从机模式，需要使用传统的 I2C 驱动 API。

### 2. 10 位地址模式

ESP-IDF 支持 10 位地址模式，配置方法如下：

```c
i2c_device_config_t dev_cfg = {
    .dev_addr_length = I2C_ADDR_BIT_LEN_10,  // 使用 10 位地址
    .device_address = 0x3FF,                 // 10 位地址示例
    .scl_speed_hz = I2C_MASTER_FREQ_HZ,
};
```

### 3. 时钟拉伸

I2C 协议支持时钟拉伸，允许从设备通过保持 SCL 线为低电平来延迟通信。ESP32 的 I2C 控制器自动支持时钟拉伸，无需额外配置。

### 4. 多主机模式

I2C 总线支持多主机模式，但需要注意以下几点：

- 确保所有主机使用相同的 SDA 和 SCL 引脚
- 避免多个主机同时发起通信
- 考虑使用仲裁机制避免冲突

## 总结

本教程详细介绍了 ESP-IDF 最新的 I2C 主机总线驱动 API 的使用方法，包括配置、基本操作、实用示例以及常见问题的解决方案。通过掌握这些知识，您可以轻松地在 ESP32 项目中集成各种 I2C 设备。

I2C 是一种非常灵活且广泛使用的通信协议，适用于连接各种低速外设。ESP32 的 I2C 控制器提供了丰富的功能和灵活的配置选项，可以满足大多数应用场景的需求。

在后续的教程中，我们将介绍更多 ESP-IDF 的功能，如 SPI、UART 等通信协议的使用方法。

## 参考资料

- [ESP-IDF 编程指南 - I2C 驱动程序](https://docs.espressif.com/projects/esp-idf/zh_CN/latest/esp32/api-reference/peripherals/i2c.html)
- [I2C 规范](https://www.nxp.com/docs/en/user-guide/UM10204.pdf)
- [ESP32 技术规格书](https://www.espressif.com/sites/default/files/documentation/esp32_datasheet_cn.pdf)

