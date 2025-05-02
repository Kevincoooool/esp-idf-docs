


          
# ESP32 USB 实现教程

## 目录
- [ESP-IDF USB 驱动实现](#esp-idf-usb-驱动实现)
- [TinyUSB 实现](#tinyusb-实现)
- [CherryUSB 实现](#cherryusb-实现)
- [常见问题](#常见问题)

## ESP-IDF USB 驱动实现

### 1. 安装 USB 驱动组件

在项目的 `CMakeLists.txt` 中添加：

```cmake
idf_component_register(
    SRCS "main.c"
    INCLUDE_DIRS "."
    REQUIRES usb
)
```

### 2. CDC 实现示例

```c
#include <stdio.h>
#include "esp_log.h"
#include "usb/usb_device.h"
#include "usb/cdc_acm_device.h"

static const char *TAG = "USB_CDC";

// CDC 事件处理函数
static void usb_cdc_cb(usb_cdc_acm_event_t *event)
{
    switch (event->type) {
        case USB_CDC_ACM_EVENT_DATA_RECEIVED:
            ESP_LOGI(TAG, "收到数据，长度: %d", event->data.len);
            // 处理接收到的数据
            break;
        case USB_CDC_ACM_EVENT_DATA_SENT:
            ESP_LOGI(TAG, "数据发送完成");
            break;
        default:
            break;
    }
}

void app_main(void)
{
    // USB 设备配置
    const usb_device_config_t dev_config = {
        .device_descriptor = NULL,  // 使用默认设备描述符
        .string_descriptor = NULL,  // 使用默认字符串描述符
        .external_phy = false,      // 使用内部 PHY
    };
    
    // 初始化 USB 设备
    ESP_ERROR_CHECK(usb_device_init(&dev_config));
    
    // CDC 配置
    usb_cdc_acm_config_t cdc_config = {
        .rx_buffer_size = 512,
        .tx_buffer_size = 512,
        .callback = usb_cdc_cb,
    };
    
    // 初始化 CDC
    usb_cdc_acm_dev_t cdc_dev;
    ESP_ERROR_CHECK(usb_cdc_acm_init(&cdc_config, &cdc_dev));
    
    // 启动 USB 设备
    ESP_ERROR_CHECK(usb_device_start());
    ESP_LOGI(TAG, "USB CDC 设备已启动");
    
    // 发送数据示例
    const char *data = "Hello USB CDC\n";
    usb_cdc_acm_write(cdc_dev, (const uint8_t *)data, strlen(data));
}
```

### 3. HID 实现示例

```c
#include <stdio.h>
#include "esp_log.h"
#include "usb/usb_device.h"
#include "usb/hid_device.h"

static const char *TAG = "USB_HID";

// HID 报告描述符（鼠标示例）
static const uint8_t hid_report_descriptor[] = {
    0x05, 0x01,  // Usage Page (Generic Desktop)
    0x09, 0x02,  // Usage (Mouse)
    0xA1, 0x01,  // Collection (Application)
    0x09, 0x01,  //   Usage (Pointer)
    0xA1, 0x00,  //   Collection (Physical)
    0x05, 0x09,  //     Usage Page (Button)
    0x19, 0x01,  //     Usage Minimum (1)
    0x29, 0x03,  //     Usage Maximum (3)
    0x15, 0x00,  //     Logical Minimum (0)
    0x25, 0x01,  //     Logical Maximum (1)
    0x95, 0x03,  //     Report Count (3)
    0x75, 0x01,  //     Report Size (1)
    0x81, 0x02,  //     Input (Data, Variable, Absolute)
    0x95, 0x01,  //     Report Count (1)
    0x75, 0x05,  //     Report Size (5)
    0x81, 0x01,  //     Input (Constant)
    0x05, 0x01,  //     Usage Page (Generic Desktop)
    0x09, 0x30,  //     Usage (X)
    0x09, 0x31,  //     Usage (Y)
    0x15, 0x81,  //     Logical Minimum (-127)
    0x25, 0x7F,  //     Logical Maximum (127)
    0x75, 0x08,  //     Report Size (8)
    0x95, 0x02,  //     Report Count (2)
    0x81, 0x06,  //     Input (Data, Variable, Relative)
    0xC0,        //   End Collection
    0xC0,        // End Collection
};

void app_main(void)
{
    // USB 设备配置
    const usb_device_config_t dev_config = {
        .device_descriptor = NULL,
        .string_descriptor = NULL,
        .external_phy = false,
    };
    
    // 初始化 USB 设备
    ESP_ERROR_CHECK(usb_device_init(&dev_config));
    
    // HID 配置
    const usb_hid_device_config_t hid_config = {
        .report_descriptor = hid_report_descriptor,
        .report_descriptor_size = sizeof(hid_report_descriptor),
        .in_report_max_len = 4,  // 鼠标报告长度
        .out_report_max_len = 0, // 不需要输出报告
    };
    
    // 初始化 HID
    usb_hid_dev_t hid_dev;
    ESP_ERROR_CHECK(usb_hid_init(&hid_config, &hid_dev));
    
    // 启动 USB 设备
    ESP_ERROR_CHECK(usb_device_start());
    ESP_LOGI(TAG, "USB HID 设备已启动");
    
    // 鼠标移动示例
    uint8_t mouse_report[4] = {0x00, 10, 10, 0};  // 向右下移动
    while (1) {
        usb_hid_report_send(hid_dev, mouse_report, sizeof(mouse_report));
        vTaskDelay(pdMS_TO_TICKS(1000));
    }
}
```

## TinyUSB 实现

### 1. 安装 TinyUSB 组件

```bash
idf.py add-dependency espressif/tinyusb
```

### 2. CDC 实现示例

```c
#include <stdio.h>
#include "esp_log.h"
#include "tinyusb.h"
#include "tusb_cdc_acm.h"

static const char *TAG = "TINYUSB_CDC";

// CDC 接收回调
void tinyusb_cdc_rx_callback(int itf, cdcacm_event_t *event)
{
    uint8_t buf[64];
    size_t rx_size = 0;
    
    // 读取接收到的数据
    esp_err_t ret = tinyusb_cdcacm_read(itf, buf, sizeof(buf), &rx_size);
    if (ret == ESP_OK) {
        ESP_LOGI(TAG, "收到数据，长度: %d", rx_size);
        // 回显数据
        tinyusb_cdcacm_write_queue(itf, buf, rx_size);
        tinyusb_cdcacm_write_flush(itf, 0);
    }
}

void app_main(void)
{
    // TinyUSB 配置
    const tinyusb_config_t tusb_cfg = {
        .device_descriptor = NULL,
        .string_descriptor = NULL,
        .external_phy = false,
    };
    
    // 初始化 TinyUSB
    ESP_ERROR_CHECK(tinyusb_driver_install(&tusb_cfg));
    
    // CDC 配置
    tinyusb_config_cdcacm_t acm_cfg = {
        .usb_dev = TINYUSB_USBDEV_0,
        .cdc_port = TINYUSB_CDC_ACM_0,
        .rx_unread_buf_sz = 64,
        .callback_rx = &tinyusb_cdc_rx_callback,
        .callback_rx_wanted_char = NULL,
        .callback_line_state_changed = NULL,
        .callback_line_coding_changed = NULL
    };
    
    // 初始化 CDC
    ESP_ERROR_CHECK(tusb_cdc_acm_init(&acm_cfg));
    ESP_LOGI(TAG, "USB CDC 设备已启动");
}
```

### 3. HID 实现示例

```c
#include <stdio.h>
#include "esp_log.h"
#include "tinyusb.h"
#include "class/hid/hid_device.h"

static const char *TAG = "TINYUSB_HID";

// HID 报告描述符（键盘示例）
const uint8_t hid_report_descriptor[] = {
    TUD_HID_REPORT_DESC_KEYBOARD()
};

void app_main(void)
{
    // TinyUSB 配置
    const tinyusb_config_t tusb_cfg = {
        .device_descriptor = NULL,
        .string_descriptor = NULL,
        .external_phy = false,
    };
    
    // 初始化 TinyUSB
    ESP_ERROR_CHECK(tinyusb_driver_install(&tusb_cfg));
    
    // HID 配置
    const tinyusb_config_hid_t hid_cfg = {
        .report_descriptor = hid_report_descriptor,
        .report_descriptor_size = sizeof(hid_report_descriptor),
        .interface_protocol = HID_ITF_PROTOCOL_KEYBOARD,
    };
    
    // 初始化 HID
    ESP_ERROR_CHECK(tinyusb_hid_init(&hid_cfg));
    ESP_LOGI(TAG, "USB HID 键盘设备已启动");
    
    // 发送按键示例
    uint8_t keycode[6] = {HID_KEY_A, 0, 0, 0, 0, 0};
    while (1) {
        // 按下 A 键
        tud_hid_keyboard_report(0, 0, keycode);
        vTaskDelay(pdMS_TO_TICKS(100));
        
        // 释放所有按键
        tud_hid_keyboard_report(0, 0, NULL);
        vTaskDelay(pdMS_TO_TICKS(1000));
    }
}
```

## CherryUSB 实现

### 1. 安装 CherryUSB 组件

```bash
idf.py add-dependency espressif/cherryusb
```

### 2. CDC 实现示例

```c
#include <stdio.h>
#include "esp_log.h"
#include "usbd_core.h"
#include "usbd_cdc.h"

static const char *TAG = "CHERRY_CDC";

// CDC 接收回调
void usbd_cdc_acm_out(uint8_t ep)
{
    uint8_t buf[64];
    uint32_t read_len = usbd_ep_read(ep, buf, sizeof(buf));
    
    if (read_len > 0) {
        ESP_LOGI(TAG, "收到数据，长度: %lu", read_len);
        // 回显数据
        usbd_ep_write(CDC_IN_EP, buf, read_len);
    }
}

void app_main(void)
{
    // 初始化 USB 设备栈
    usbd_init();
    
    // 注册 CDC 类
    usbd_add_interface(usbd_cdc_acm_init_intf());
    
    // 启动 USB 设备
    usbd_start();
    ESP_LOGI(TAG, "USB CDC 设备已启动");
    
    // 发送数据示例
    const char *data = "Hello CherryUSB CDC\n";
    while (1) {
        usbd_ep_write(CDC_IN_EP, (uint8_t *)data, strlen(data));
        vTaskDelay(pdMS_TO_TICKS(1000));
    }
}
```

### 3. HID 实现示例

```c
#include <stdio.h>
#include "esp_log.h"
#include "usbd_core.h"
#include "usbd_hid.h"

static const char *TAG = "CHERRY_HID";

// HID 报告描述符（游戏手柄示例）
const uint8_t hid_report_descriptor[] = {
    0x05, 0x01,        // Usage Page (Generic Desktop)
    0x09, 0x05,        // Usage (Game Pad)
    0xA1, 0x01,        // Collection (Application)
    0x09, 0x01,        //   Usage (Pointer)
    0xA1, 0x00,        //   Collection (Physical)
    0x09, 0x30,        //     Usage (X)
    0x09, 0x31,        //     Usage (Y)
    0x15, 0x00,        //     Logical Minimum (0)
    0x26, 0xFF, 0x00,  //     Logical Maximum (255)
    0x75, 0x08,        //     Report Size (8)
    0x95, 0x02,        //     Report Count (2)
    0x81, 0x02,        //     Input (Data, Variable, Absolute)
    0x05, 0x09,        //     Usage Page (Button)
    0x19, 0x01,        //     Usage Minimum (Button 1)
    0x29, 0x08,        //     Usage Maximum (Button 8)
    0x15, 0x00,        //     Logical Minimum (0)
    0x25, 0x01,        //     Logical Maximum (1)
    0x75, 0x01,        //     Report Size (1)
    0x95, 0x08,        //     Report Count (8)
    0x81, 0x02,        //     Input (Data, Variable, Absolute)
    0xC0,              //   End Collection
    0xC0               // End Collection
};

void app_main(void)
{
    // 初始化 USB 设备栈
    usbd_init();
    
    // HID 配置
    struct usbd_interface *hid_intf = usbd_hid_init_intf(hid_report_descriptor,
                                                        sizeof(hid_report_descriptor));
    
    // 注册 HID 类
    usbd_add_interface(hid_intf);
    
    // 启动 USB 设备
    usbd_start();
    ESP_LOGI(TAG, "USB HID 游戏手柄设备已启动");
    
    // 发送游戏手柄数据示例
    uint8_t report[3] = {128, 128, 0x00};  // X=128, Y=128, 无按钮按下
    while (1) {
        usbd_ep_write(HID_IN_EP, report, sizeof(report));
        vTaskDelay(pdMS_TO_TICKS(1000));
    }
}
```

## 常见问题

### 1. USB 连接问题

常见原因和解决方法：
- 检查 USB 线缆质量
- 确认 USB 端口供电充足
- 验证驱动程序安装正确
- 检查设备描述符配置

### 2. 数据传输问题

解决方案：
- 合理设置缓冲区大小
- 正确处理数据包大小
- 实现适当的错误处理
- 添加数据校验机制

### 3. 性能优化

建议：
- 使用 DMA 传输
- 优化缓冲区管理
- 实现数据包合并
- 减少中断处理时间

## 总结

本教程介绍了在 ESP32-S2/S3 上实现 USB 功能的三种方式：

1. **ESP-IDF USB 驱动**：
   - 官方支持
   - 基本功能完整
   - 配置简单

2. **TinyUSB**：
   - 跨平台支持
   - 功能丰富
   - 社区活跃

3. **CherryUSB**：
   - 轻量级实现
   - 性能优良
   - 配置灵活

选择建议：
- 简单应用使用 ESP-IDF USB 驱动
- 复杂功能使用 TinyUSB
- 追求性能使用 CherryUSB

## 参考资料

- [ESP-IDF USB 文档](https://docs.espressif.com/projects/esp-idf/zh_CN/latest/esp32s2/api-reference/peripherals/usb_device.html)
- [TinyUSB 文档](https://docs.tinyusb.org/)
- [CherryUSB 文档](https://cherryusb.readthedocs.io/)

        


          
# ESP32-S3 USB硬件连接指南

## USB引脚说明

ESP32-S3内置USB 1.1 OTG控制器和PHY,支持两种USB连接方式:

### 1. 内置USB PHY
- **D+**: GPIO19
- **D-**: GPIO20
- 无需外部晶振和PHY芯片
- 默认配置方式,推荐使用

### 2. 外部USB PHY 
- **D+**: 可配置任意GPIO
- **D-**: 可配置任意GPIO
- 需要外部USB PHY芯片
- 需要在初始化时配置external_phy = true

## 硬件连接示例

### 1. 使用内置PHY的最小系统

```
ESP32-S3                   USB连接器
┌──────────┐              ┌─────────┐
│          │              │         │
│    GPIO19├──────────────┤D+       │
│    GPIO20├──────────────┤D-       │
│      GND ├──────────────┤GND      │
│          │              │         │
└──────────┘              └─────────┘
```

### 2. 带ESD保护的推荐电路

```
ESP32-S3            ESD保护器          USB连接器
┌──────────┐      ┌──────────┐      ┌─────────┐
│          │      │          │      │         │
│    GPIO19├──────┤D+ IN  D+├──────┤D+       │
│    GPIO20├──────┤D- IN  D-├──────┤D-       │
│      GND ├──────┤GND   GND├──────┤GND      │
│          │      │          │      │         │
└──────────┘      └──────────┘      └─────────┘
```

## 硬件设计注意事项

1. **信号完整性**
   - D+/D-走线要等长
   - 走线长度尽量短
   - 避免过孔和拐角
   - 保持90Ω阻抗

2. **电源要求**
   - USB供电电压:5V±5%
   - 工作电流:≤500mA
   - 建议加去耦电容

3. **ESD保护**
   - 推荐使用专用USB ESD保护器
   - 常用型号:USBLC6-2SC6
   - 保护二极管要靠近USB接口

4. **PCB布局布线**
   - D+/D-差分对紧密布线
   - 远离干扰源
   - 地平面完整
   - 加保护接地环

## 配置说明

在使用内置PHY时,需要在USB初始化配置中设置:

```c
const usb_device_config_t dev_config = {
    .device_descriptor = NULL,
    .string_descriptor = NULL,
    .external_phy = false,  // 使用内置PHY
};
```

如果使用外部PHY,则需要:

```c
const usb_device_config_t dev_config = {
    .device_descriptor = NULL,
    .string_descriptor = NULL,
    .external_phy = true,   // 使用外部PHY
    .gpio_conf = {
        .dp_pin = GPIO_NUM_X,  // 自定义D+引脚
        .dm_pin = GPIO_NUM_Y,  // 自定义D-引脚
    }
};
```

## 调试建议

1. 使用USB分析仪监测信号质量
2. 测量D+/D-电压波形
3. 检查USB枚举过程
4. 验证供电电压稳定性

## 常见问题

1. **无法识别设备**
   - 检查D+/D-连接是否正确
   - 确认供电电压
   - 验证固件配置

2. **通信不稳定**
   - 检查信号完整性
   - 增加ESD保护
   - 优化PCB布局

3. **供电问题**
   - 测量USB供电电压
   - 检查电流消耗
   - 增加电源滤波

        