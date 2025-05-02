


          
# ESP32 LVGL 开发教程

## 1. LVGL 简介

LVGL（Light and Versatile Graphics Library）是一个免费开源的图形库，提供了创建嵌入式GUI所需的一切，具有易用的图形元素、美观的视觉效果和低内存占用的特点。它是专为微控制器和低功耗系统设计的，但同样适用于更高性能的平台。

### 1.1 LVGL 的主要特点

- **轻量级**：核心库占用较小，适合资源受限的嵌入式系统
- **丰富的组件**：按钮、标签、滑块、图表等多种UI组件
- **美观的视觉效果**：支持动画、透明度、平滑渐变等效果
- **多平台支持**：可在各种微控制器和操作系统上运行
- **多语言支持**：支持UTF-8编码的Unicode字符

## 2. 项目结构解析

本教程基于ESP32平台，使用LVGL v9版本。让我们先了解项目的主要文件结构：

### 2.1 主要文件

- **app_main.c**：应用程序入口，初始化LVGL并运行示例
- **ksdiy_lvgl_port.c**：LVGL与ESP32硬件的接口实现
- **ksdiy_lvgl_port.h**：接口函数声明
- **idf_component.yml**：项目依赖配置
- **CMakeLists.txt**：编译配置文件

### 2.2 依赖组件

从`idf_component.yml`可以看出，项目依赖以下组件：

```yaml
dependencies:
  idf: ">=4.4"
  lvgl/lvgl: "^9.2.2"
  esp_lcd_touch_cst816s: "^1.0"
  esp_io_expander_tca9554: "^2.0.0"
  espressif/esp_lvgl_port: "^2.3.0"
```

这些依赖包括：
- ESP-IDF框架（版本4.4及以上）
- LVGL库（版本9.2.2）
- CST816S触摸屏驱动
- TCA9554 IO扩展芯片驱动
- ESP LVGL端口库

## 3. LVGL初始化流程

让我们通过代码来理解LVGL的初始化流程：

### 3.1 应用程序入口

`app_main.c`是程序的入口点：

```c
void app_main(void)
{
    ksdiy_lvgl_port_init();    // 初始化LVGL端口
    ksdiy_lvgl_lock(-1);       // 获取LVGL互斥锁
    lv_demo_music();           // 运行音乐播放器演示
    ksdiy_lvgl_unlock();       // 释放LVGL互斥锁
}
```

这个简单的入口函数完成了三个主要步骤：
1. 初始化LVGL及其硬件接口
2. 获取互斥锁（LVGL不是线程安全的）
3. 启动音乐播放器演示程序

### 3.2 LVGL端口初始化

`ksdiy_lvgl_port_init()`函数是整个初始化过程的核心，它完成了以下工作：

#### 3.2.1 I2C初始化

```c
i2c_init();
```

初始化I2C总线，用于与触摸屏和IO扩展芯片通信。

#### 3.2.2 IO扩展芯片初始化

```c
// 初始化扩展IO芯片 先找TCA9554 如果找不到就找TCA9554A
if ((esp_io_expander_new_i2c_tca9554(touch_i2c_bus_, BSP_IO_EXPANDER_I2C_ADDRESS_TCA9554, &io_expander) != ESP_OK))
{
    if (esp_io_expander_new_i2c_tca9554(touch_i2c_bus_, BSP_IO_EXPANDER_I2C_ADDRESS_TCA9554A, &io_expander) != ESP_OK)
    {
        ESP_LOGE(TAG, "Failed to initialize IO expander");
    }
    else
    {
        // 配置IO扩展芯片
        // ...
    }
}
```

这段代码尝试初始化TCA9554或TCA9554A IO扩展芯片，用于扩展GPIO引脚。

#### 3.2.3 SPI总线初始化

```c
ESP_LOGI(TAG, "Initialize SPI bus");
spi_bus_config_t buscfg = {
    .sclk_io_num = KSDIY_PIN_NUM_SCLK,
    .mosi_io_num = KSDIY_PIN_NUM_MOSI,
    .miso_io_num = KSDIY_PIN_NUM_MISO,
    .quadwp_io_num = -1,
    .quadhd_io_num = -1,
    .max_transfer_sz = KSDIY_LCD_H_RES * 240 * sizeof(uint16_t),
};
ESP_ERROR_CHECK(spi_bus_initialize(LCD_HOST, &buscfg, SPI_DMA_CH_AUTO));
```

初始化SPI总线，用于与LCD显示屏通信。

#### 3.2.4 LCD面板初始化

```c
// 配置LCD面板IO
esp_lcd_panel_io_handle_t io_handle = NULL;
esp_lcd_panel_io_spi_config_t io_config = {
    .dc_gpio_num = KSDIY_PIN_NUM_LCD_DC,
    .cs_gpio_num = KSDIY_PIN_NUM_LCD_CS,
    .pclk_hz = KSDIY_LCD_PIXEL_CLOCK_HZ,
    .lcd_cmd_bits = KSDIY_LCD_CMD_BITS,
    .lcd_param_bits = KSDIY_LCD_PARAM_BITS,
    .spi_mode = 0,
    .trans_queue_depth = 10,
};

// 安装ST7789面板驱动
ESP_ERROR_CHECK(esp_lcd_new_panel_st7789(io_handle, &panel_config, &panel_handle));
ESP_ERROR_CHECK(esp_lcd_panel_reset(panel_handle));
ESP_ERROR_CHECK(esp_lcd_panel_init(panel_handle));
// 配置显示方向等
ESP_ERROR_CHECK(esp_lcd_panel_swap_xy(panel_handle, true));
ESP_ERROR_CHECK(esp_lcd_panel_mirror(panel_handle, false, true));
ESP_ERROR_CHECK(esp_lcd_panel_invert_color(panel_handle, true));
ESP_ERROR_CHECK(esp_lcd_panel_disp_on_off(panel_handle, true));
```

这部分代码初始化LCD面板，配置SPI接口，并设置显示方向、镜像等参数。

#### 3.2.5 触摸屏初始化

```c
app_touch_init();
```

初始化触摸屏控制器，配置I2C接口和触摸参数。

#### 3.2.6 LVGL库初始化

```c
ESP_LOGI(TAG, "Initialize LVGL library");
lv_init();

// 分配LVGL缓冲区
uint8_t color_bytes = lv_color_format_get_size(LV_COLOR_FORMAT_RGB565);
void *buf1 = heap_caps_malloc(KSDIY_LCD_H_RES * 10 * color_bytes, MALLOC_CAP_DMA);
void *buf2 = heap_caps_malloc(KSDIY_LCD_H_RES * 10 * color_bytes, MALLOC_CAP_DMA);

// 注册显示设备
disp = lv_display_create(KSDIY_LCD_H_RES, KSDIY_LCD_V_RES);
lv_display_set_buffers(disp, buf1, buf2, KSDIY_LCD_H_RES * 10 * color_bytes, LV_DISPLAY_RENDER_MODE_PARTIAL);
lv_display_set_user_data(disp, panel_handle);
lv_display_set_flush_cb(disp, lcd_lvgl_flush_cb);
lv_display_set_color_format(disp, LV_COLOR_FORMAT_RGB565);
```

这部分代码初始化LVGL库，分配显示缓冲区，并注册显示设备。

#### 3.2.7 LVGL定时器和任务创建

```c
// 创建LVGL定时器
const esp_timer_create_args_t lvgl_tick_timer_args = {
    .callback = &ksdiy_increase_lvgl_tick,
    .name = "lvgl_tick"
};
esp_timer_handle_t lvgl_tick_timer = NULL;
ESP_ERROR_CHECK(esp_timer_create(&lvgl_tick_timer_args, &lvgl_tick_timer));
ESP_ERROR_CHECK(esp_timer_start_periodic(lvgl_tick_timer, EXAMPLE_LVGL_TICK_PERIOD_MS * 1000));

// 创建互斥锁
lvgl_mux = xSemaphoreCreateRecursiveMutex();
assert(lvgl_mux);

// 创建LVGL任务
xTaskCreate(ksdiy_lvgl_port_task, "LVGL", EXAMPLE_LVGL_TASK_STACK_SIZE, NULL, EXAMPLE_LVGL_TASK_PRIORITY, NULL);
```

最后，创建LVGL定时器用于计时，创建互斥锁保证线程安全，并创建LVGL任务处理GUI更新。

## 4. LVGL任务处理

LVGL需要定期调用`lv_timer_handler()`函数来处理GUI更新。在本项目中，通过创建一个单独的任务来实现：

```c
static void ksdiy_lvgl_port_task(void *arg)
{
    ESP_LOGI(TAG, "Starting LVGL task");
    uint32_t task_delay_ms = EXAMPLE_LVGL_TASK_MAX_DELAY_MS;
    while (1)
    {
        // 获取互斥锁
        if (ksdiy_lvgl_lock(-1))
        {
            task_delay_ms = lv_timer_handler();
            // 释放互斥锁
            ksdiy_lvgl_unlock();
        }
        
        // 限制任务延迟时间
        if (task_delay_ms > EXAMPLE_LVGL_TASK_MAX_DELAY_MS)
        {
            task_delay_ms = EXAMPLE_LVGL_TASK_MAX_DELAY_MS;
        }
        else if (task_delay_ms < EXAMPLE_LVGL_TASK_MIN_DELAY_MS)
        {
            task_delay_ms = EXAMPLE_LVGL_TASK_MIN_DELAY_MS;
        }
        
        vTaskDelay(pdMS_TO_TICKS(task_delay_ms));
    }
}
```

这个任务在一个无限循环中执行以下操作：
1. 获取互斥锁
2. 调用`lv_timer_handler()`处理GUI更新
3. 释放互斥锁
4. 根据返回值延时一段时间

## 5. 显示刷新回调

LVGL需要知道如何将图形数据发送到实际的显示设备，这通过刷新回调函数实现：

```c
static void lcd_lvgl_flush_cb(lv_display_t *drv, const lv_area_t *area, unsigned char *color_map)
{
    esp_lcd_panel_handle_t panel_handle = (esp_lcd_panel_handle_t)lv_display_get_user_data(drv);

    int offsetx1 = area->x1 + offset_x;
    int offsetx2 = area->x2 + offset_x;
    int offsety1 = area->y1;
    int offsety2 = area->y2;
    
    // 交换RGB565格式的字节顺序
    size_t len = lv_area_get_size(area);
    lv_draw_sw_rgb565_swap(color_map, len);
    
    // 将数据发送到LCD面板
    esp_lcd_panel_draw_bitmap(panel_handle, offsetx1, offsety1, offsetx2 + 1, offsety2 + 1, color_map);
}
```

这个回调函数负责将LVGL生成的图形数据发送到LCD显示屏。

## 6. 互斥锁机制

由于LVGL不是线程安全的，需要使用互斥锁来保护LVGL API的调用：

```c
bool ksdiy_lvgl_lock(int timeout_ms)
{
    // 将超时时间转换为FreeRTOS滴答数
    // 如果timeout_ms设置为-1，程序将阻塞直到条件满足
    const TickType_t timeout_ticks = (timeout_ms == -1) ? portMAX_DELAY : pdMS_TO_TICKS(timeout_ms);
    return xSemaphoreTakeRecursive(lvgl_mux, timeout_ticks) == pdTRUE;
}

void ksdiy_lvgl_unlock(void)
{
    xSemaphoreGiveRecursive(lvgl_mux);
}
```

这两个函数提供了获取和释放互斥锁的接口，确保在多任务环境中安全地调用LVGL API。

## 7. 创建简单的LVGL应用

现在我们了解了LVGL的初始化和运行机制，让我们看看如何创建一个简单的LVGL应用。

### 7.1 创建一个简单的按钮

```c
void create_simple_button(void)
{
    // 获取LVGL互斥锁
    ksdiy_lvgl_lock(-1);
    
    // 创建一个按钮
    lv_obj_t *btn = lv_btn_create(lv_scr_act());
    lv_obj_set_size(btn, 120, 50);
    lv_obj_align(btn, LV_ALIGN_CENTER, 0, 0);
    
    // 创建一个标签放在按钮上
    lv_obj_t *label = lv_label_create(btn);
    lv_label_set_text(label, "按钮");
    lv_obj_align(label, LV_ALIGN_CENTER, 0, 0);
    
    // 释放LVGL互斥锁
    ksdiy_lvgl_unlock();
}
```

这个简单的例子创建了一个居中的按钮，上面有一个"按钮"文本标签。

### 7.2 添加事件处理

```c
static void btn_event_cb(lv_event_t *e)
{
    lv_event_code_t code = lv_event_get_code(e);
    if (code == LV_EVENT_CLICKED) {
        // 按钮被点击时的处理
        static uint8_t cnt = 0;
        cnt++;
        
        // 获取标签对象
        lv_obj_t *btn = lv_event_get_target(e);
        lv_obj_t *label = lv_obj_get_child(btn, 0);
        
        // 更新标签文本
        char buf[32];
        sprintf(buf, "点击次数: %d", cnt);
        lv_label_set_text(label, buf);
    }
}

void create_button_with_event(void)
{
    ksdiy_lvgl_lock(-1);
    
    // 创建按钮
    lv_obj_t *btn = lv_btn_create(lv_scr_act());
    lv_obj_set_size(btn, 150, 50);
    lv_obj_align(btn, LV_ALIGN_CENTER, 0, 0);
    lv_obj_add_event_cb(btn, btn_event_cb, LV_EVENT_CLICKED, NULL);
    
    // 创建标签
    lv_obj_t *label = lv_label_create(btn);
    lv_label_set_text(label, "点击我");
    lv_obj_align(label, LV_ALIGN_CENTER, 0, 0);
    
    ksdiy_lvgl_unlock();
}
```

这个例子创建了一个带有事件处理的按钮，每次点击都会更新显示的计数。

## 8. 常见问题与解决方案

### 8.1 显示屏不显示或显示异常

- 检查SPI配置是否正确
- 确认LCD型号与驱动匹配
- 检查显示方向设置（swap_xy, mirror_x, mirror_y）
- 确认背光控制引脚配置正确

### 8.2 触摸屏不响应

- 检查I2C配置是否正确
- 确认触摸屏型号与驱动匹配
- 检查触摸坐标转换设置（swap_xy, mirror_x, mirror_y）
- 使用I2C扫描工具确认触摸芯片地址

### 8.3 内存不足

- 减小LVGL缓冲区大小
- 禁用不需要的LVGL功能
- 优化任务堆栈大小
- 使用ESP32的外部PSRAM（如果有）

## 9. 进阶主题

### 9.1 自定义字体

LVGL支持自定义字体，可以使用在线工具将TTF字体转换为LVGL兼容格式。

### 9.2 图片显示

LVGL支持多种图片格式，包括静态图片和动画。可以将图片转换为C数组或使用文件系统加载。

### 9.3 主题定制

LVGL提供了主题系统，可以统一定制UI的外观。

## 10. 总结

本教程介绍了在ESP32平台上使用LVGL v9开发图形界面的基本流程，包括：

1. LVGL的初始化与配置
2. 显示屏和触摸屏的驱动集成
3. 多任务环境下的LVGL使用
4. 简单UI组件的创建与事件处理

通过理解这些基础知识，你可以开始创建自己的图形界面应用。LVGL提供了丰富的组件和功能，可以满足大多数嵌入式GUI的需求。

## 11. 参考资源

- [LVGL官方文档](https://docs.lvgl.io/)
- [ESP-IDF编程指南](https://docs.espressif.com/projects/esp-idf/en/latest/)
- [LVGL示例代码](https://github.com/lvgl/lvgl/tree/master/examples)

希望本教程能帮助你快速入门LVGL开发！
