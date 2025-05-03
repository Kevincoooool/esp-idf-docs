


          

# ESP32-S3 KSDIY Korvo 开发板 LVGL 与 JPG 下载显示教程

## 1. 项目概述

本教程将介绍如何使用 ESP32-S3 KSDIY Korvo 开发板实现 WiFi 连接、JPG 图片下载和显示功能。项目基于 ESP-IDF 框架和 LVGL 图形库，适合初学者学习嵌入式系统开发。

## 2. 硬件准备

- KSDIY Korvo 开发板（基于 ESP32-S3）
- LCD 显示屏（项目中使用 280x240 分辨率）
- 触摸屏（CST816S 触摸控制器）
- USB 数据线

## 3. 软件环境

- ESP-IDF 开发环境
- LVGL 图形库 v8.3.0
- ESP JPEG 解码库

## 4. 项目结构

项目主要包含以下几个部分：

1. **WiFi 连接模块**：负责连接到指定的 WiFi 网络
2. **HTTP 下载模块**：从网络下载 JPG 图片
3. **JPEG 解码模块**：将下载的 JPG 图片解码为 RGB565 格式
4. **LVGL 显示模块**：在 LCD 上显示解码后的图片

## 5. 代码详解

### 5.1 主程序 (app_main.c)

主程序负责初始化各个模块并协调它们的工作：

```c
void app_main(void)
{
    // 初始化 NVS（非易失性存储）
    esp_err_t ret = nvs_flash_init();
    if (ret == ESP_ERR_NVS_NO_FREE_PAGES)
    {
        ESP_ERROR_CHECK(nvs_flash_erase());
        ret = nvs_flash_init();
    }
    ESP_ERROR_CHECK(ret);

    // 初始化 WiFi 并连接到网络
    wifi_init_sta();

    // 分配内存用于图像处理
    img_rgb565 = heap_caps_malloc(800 * 480 * 2, MALLOC_CAP_8BIT | MALLOC_CAP_SPIRAM);
    pbuffer = heap_caps_malloc(500 * 1024, MALLOC_CAP_8BIT | MALLOC_CAP_SPIRAM);

    // 下载 JPG 图片
    http_download_jpg("http://i2.hdslb.com/bfs/face/bce14f5e3af4bca480fc7de227986ba304507078.jpg");
    
    // 初始化 LVGL 图形库
    ksdiy_lvgl_port_init();

    // 创建并显示图像
    if (ksdiy_lvgl_lock(0))
    {
        img_jpg = lv_img_create(lv_scr_act());
        lv_obj_set_pos(img_jpg, 20, 20);
        
        if (Jpgsize > 0)
        {
            // 解码 JPG 图片
            esp_jpeg_decode_one_picture((uint8_t *)face_buffer, Jpgsize, &img_rgb565, &Rgbsize);
            
            // 设置图像属性
            jpg_data.header.w = rgb_width;
            jpg_data.header.h = rgb_height;
            jpg_data.data_size = rgb_width * rgb_height * 2;
            jpg_data.data = (uint8_t *)img_rgb565;
            
            // 显示图像
            lv_img_set_src(img_jpg, &jpg_data);
            lv_obj_align(img_jpg, LV_ALIGN_CENTER, 60, 0);
        }
        
        ksdiy_lvgl_unlock();
    }
}
```

### 5.2 WiFi 连接模块 (app_wifi.c)

WiFi 模块负责连接到指定的无线网络：

```c
void wifi_init_sta(void)
{
    // 创建 FreeRTOS 事件组
    s_wifi_event_group = xEventGroupCreate();

    // 初始化网络接口
    ESP_ERROR_CHECK(esp_netif_init());
    ESP_ERROR_CHECK(esp_event_loop_create_default());
    esp_netif_create_default_wifi_sta();

    // 配置 WiFi
    wifi_init_config_t cfg = WIFI_INIT_CONFIG_DEFAULT();
    ESP_ERROR_CHECK(esp_wifi_init(&cfg));

    // 注册事件处理函数
    ESP_ERROR_CHECK(esp_event_handler_instance_register(WIFI_EVENT,
                                                        ESP_EVENT_ANY_ID,
                                                        &event_handler,
                                                        NULL,
                                                        &instance_any_id));
    ESP_ERROR_CHECK(esp_event_handler_instance_register(IP_EVENT,
                                                        IP_EVENT_STA_GOT_IP,
                                                        &event_handler,
                                                        NULL,
                                                        &instance_got_ip));

    // 设置 WiFi 连接参数
    wifi_config_t wifi_config = {
        .sta = {
            .ssid = EXAMPLE_ESP_WIFI_SSID,
            .password = EXAMPLE_ESP_WIFI_PASS,
            .threshold.authmode = ESP_WIFI_SCAN_AUTH_MODE_THRESHOLD,
            .sae_pwe_h2e = ESP_WIFI_SAE_MODE,
            .sae_h2e_identifier = EXAMPLE_H2E_IDENTIFIER,
        },
    };
    
    // 启动 WiFi
    ESP_ERROR_CHECK(esp_wifi_set_mode(WIFI_MODE_STA));
    ESP_ERROR_CHECK(esp_wifi_set_config(WIFI_IF_STA, &wifi_config));
    ESP_ERROR_CHECK(esp_wifi_start());

    ESP_LOGI(TAG, "wifi_init_sta finished.");

    // 等待连接成功或失败
    EventBits_t bits = xEventGroupWaitBits(s_wifi_event_group,
            WIFI_CONNECTED_BIT | WIFI_FAIL_BIT,
            pdFALSE,
            pdFALSE,
            portMAX_DELAY);

    // 输出连接结果
    if (bits & WIFI_CONNECTED_BIT) {
        ESP_LOGI(TAG, "connected to ap SSID:%s password:%s",
                 EXAMPLE_ESP_WIFI_SSID, EXAMPLE_ESP_WIFI_PASS);
    } else if (bits & WIFI_FAIL_BIT) {
        ESP_LOGI(TAG, "Failed to connect to SSID:%s, password:%s",
                 EXAMPLE_ESP_WIFI_SSID, EXAMPLE_ESP_WIFI_PASS);
    } else {
        ESP_LOGE(TAG, "UNEXPECTED EVENT");
    }
}
```

### 5.3 JPEG 解码模块

JPEG 解码模块负责将 JPG 图片转换为 RGB565 格式，以便在 LCD 上显示：

```c
jpeg_error_t esp_jpeg_decode_one_picture(uint8_t *input_buf, int len, uint8_t **output_buf, int *out_len)
{
    uint8_t *out_buf = NULL;
    jpeg_error_t ret = JPEG_ERR_OK;
    jpeg_dec_io_t *jpeg_io = NULL;
    jpeg_dec_header_info_t *out_info = NULL;

    // 生成默认配置
    jpeg_dec_config_t config = DEFAULT_JPEG_DEC_CONFIG();
    config.output_type = j_type;  // RGB565_BE 格式
    config.rotate = j_rotation;   // 不旋转

    // 创建 JPEG 解码器句柄
    jpeg_dec_handle_t jpeg_dec = NULL;
    ret = jpeg_dec_open(&config, &jpeg_dec);
    if (ret != JPEG_ERR_OK) {
        return ret;
    }

    // 创建 IO 回调句柄
    jpeg_io = calloc(1, sizeof(jpeg_dec_io_t));
    if (jpeg_io == NULL) {
        ret = JPEG_ERR_NO_MEM;
        goto jpeg_dec_failed;
    }

    // 创建输出信息句柄
    out_info = calloc(1, sizeof(jpeg_dec_header_info_t));
    if (out_info == NULL) {
        ret = JPEG_ERR_NO_MEM;
        goto jpeg_dec_failed;
    }

    // 设置输入缓冲区
    jpeg_io->inbuf = input_buf;
    jpeg_io->inbuf_len = len;

    // 解析 JPEG 头部
    ret = jpeg_dec_parse_header(jpeg_dec, jpeg_io, out_info);
    if (ret != JPEG_ERR_OK) {
        goto jpeg_dec_failed;
    }
    
    // 获取图像尺寸
    rgb_width = out_info->width;
    rgb_height = out_info->height;
    ESP_LOGI(TAG, "img width:%d height:%d ", rgb_width, rgb_height);

    // 计算输出缓冲区大小
    *out_len = out_info->width * out_info->height * 3;
    if (config.output_type == JPEG_PIXEL_FORMAT_RGB565_LE || 
        config.output_type == JPEG_PIXEL_FORMAT_RGB565_BE || 
        config.output_type == JPEG_PIXEL_FORMAT_CbYCrY) {
        *out_len = out_info->width * out_info->height * 2;
    } else if (config.output_type == JPEG_PIXEL_FORMAT_RGB888) {
        *out_len = out_info->width * out_info->height * 3;
    } else {
        ret = JPEG_ERR_INVALID_PARAM;
        goto jpeg_dec_failed;
    }
    
    // 分配输出缓冲区
    out_buf = jpeg_calloc_align(*out_len, 16);
    if (out_buf == NULL) {
        ret = JPEG_ERR_NO_MEM;
        goto jpeg_dec_failed;
    }
    jpeg_io->outbuf = out_buf;
    *output_buf = out_buf;

    // 开始解码 JPEG
    ret = jpeg_dec_process(jpeg_dec, jpeg_io);
    if (ret != JPEG_ERR_OK) {
        goto jpeg_dec_failed;
    }

    // 解码器清理
jpeg_dec_failed:
    jpeg_dec_close(jpeg_dec);
    if (jpeg_io) {
        free(jpeg_io);
    }
    if (out_info) {
        free(out_info);
    }
    return ret;
}
```

### 5.4 LVGL 显示模块 (ksdiy_lvgl_port.c)

LVGL 显示模块负责初始化 LCD 和触摸屏，并提供图形界面：

```c
void ksdiy_lvgl_port_init(void)
{
    // 创建互斥锁，用于 LVGL 线程安全
    lvgl_mux = xSemaphoreCreateRecursiveMutex();
    assert(lvgl_mux);

    // 初始化 I2C 总线（用于触摸屏）
    i2c_init();

    // 初始化 LCD 控制器
    ESP_LOGI(TAG, "Initialize SPI bus");
    spi_bus_config_t buscfg = {
        .sclk_io_num = KSDIY_PIN_NUM_SCLK,
        .mosi_io_num = KSDIY_PIN_NUM_MOSI,
        .miso_io_num = KSDIY_PIN_NUM_MISO,
        .quadwp_io_num = -1,
        .quadhd_io_num = -1,
        .max_transfer_sz = EXAMPLE_LCD_H_RES * 80 * sizeof(uint16_t),
    };
    ESP_ERROR_CHECK(spi_bus_initialize(LCD_HOST, &buscfg, SPI_DMA_CH_AUTO));

    // 配置 LCD 面板 IO
    ESP_LOGI(TAG, "Initialize panel IO");
    esp_lcd_panel_io_handle_t io_handle = NULL;
    esp_lcd_panel_io_spi_config_t io_config = {
        .dc_gpio_num = KSDIY_PIN_NUM_LCD_DC,
        .cs_gpio_num = KSDIY_PIN_NUM_LCD_CS,
        .pclk_hz = EXAMPLE_LCD_PIXEL_CLOCK_HZ,
        .lcd_cmd_bits = EXAMPLE_LCD_CMD_BITS,
        .lcd_param_bits = EXAMPLE_LCD_PARAM_BITS,
        .spi_mode = 0,
        .trans_queue_depth = 10,
        .on_color_trans_done = ksdiy_notify_lvgl_flush_ready,
        .user_ctx = &disp_drv,
    };
    ESP_ERROR_CHECK(esp_lcd_new_panel_io_spi((esp_lcd_spi_bus_handle_t)LCD_HOST, &io_config, &io_handle));

    // 配置 LCD 面板
    ESP_LOGI(TAG, "Initialize LCD panel");
    esp_lcd_panel_handle_t panel_handle = NULL;
    esp_lcd_panel_dev_config_t panel_config = {
        .reset_gpio_num = KSDIY_PIN_NUM_LCD_RST,
        .rgb_endian = LCD_RGB_ENDIAN_RGB,
        .bits_per_pixel = 16,
    };
    ESP_ERROR_CHECK(esp_lcd_new_panel_st7789(io_handle, &panel_config, &panel_handle));

    // 初始化 LCD
    esp_lcd_panel_reset(panel_handle);
    esp_lcd_panel_init(panel_handle);
    esp_lcd_panel_invert_color(panel_handle, true);
    esp_lcd_panel_swap_xy(panel_handle, false);
    esp_lcd_panel_mirror(panel_handle, true, false);
    esp_lcd_panel_set_gap(panel_handle, 0, 0);

    // 初始化 LVGL 库
    ESP_LOGI(TAG, "Initialize LVGL library");
    lv_init();

    // 分配绘图缓冲区
    void *buf1 = NULL;
    void *buf2 = NULL;
    buf1 = heap_caps_malloc(EXAMPLE_LCD_H_RES * 100 * sizeof(lv_color_t), MALLOC_CAP_DMA);
    assert(buf1);
    buf2 = heap_caps_malloc(EXAMPLE_LCD_H_RES * 100 * sizeof(lv_color_t), MALLOC_CAP_DMA);
    assert(buf2);

    // 初始化 LVGL 绘图缓冲区
    lv_disp_draw_buf_init(&disp_buf, buf1, buf2, EXAMPLE_LCD_H_RES * 100);

    // 注册显示驱动
    ESP_LOGI(TAG, "Register display driver to LVGL");
    lv_disp_drv_init(&disp_drv);
    disp_drv.hor_res = EXAMPLE_LCD_H_RES;
    disp_drv.ver_res = EXAMPLE_LCD_V_RES;
    disp_drv.flush_cb = ksdiy_lvgl_flush_cb;
    disp_drv.draw_buf = &disp_buf;
    disp_drv.user_data = panel_handle;
    disp_drv.drv_update_cb = ksdiy_lvgl_port_update_callback;
    disp = lv_disp_drv_register(&disp_drv);

    // 初始化触摸屏
#if CONFIG_EXAMPLE_LCD_TOUCH_ENABLED
    app_touch_init();
    if (cat816s_found == true)
    {
        ESP_LOGI(TAG, "Register touch driver to LVGL");
        lv_indev_drv_init(&indev_drv);
        indev_drv.type = LV_INDEV_TYPE_POINTER;
        indev_drv.read_cb = ksdiy_lvgl_touch_cb;
        indev_drv.user_data = tp;
        lv_indev_drv_register(&indev_drv);
    }
#endif

    // 创建 LVGL 定时器
    ESP_LOGI(TAG, "Install LVGL tick timer");
    const esp_timer_create_args_t lvgl_tick_timer_args = {
        .callback = &ksdiy_increase_lvgl_tick,
        .name = "lvgl_tick"
    };
    esp_timer_handle_t lvgl_tick_timer = NULL;
    ESP_ERROR_CHECK(esp_timer_create(&lvgl_tick_timer_args, &lvgl_tick_timer));
    ESP_ERROR_CHECK(esp_timer_start_periodic(lvgl_tick_timer, EXAMPLE_LVGL_TICK_PERIOD_MS * 1000));

    // 创建 LVGL 任务
    ESP_LOGI(TAG, "Create LVGL task");
    xTaskCreatePinnedToCore(ksdiy_lvgl_port_task, "lvgl", EXAMPLE_LVGL_TASK_STACK_SIZE, NULL, EXAMPLE_LVGL_TASK_PRIORITY, NULL, 1);
}
```

## 6. 配置说明

### 6.1 WiFi 配置 (Kconfig.projbuild)

通过 menuconfig 可以配置 WiFi 连接参数：

```
menu "Example Configuration"

    config ESP_WIFI_SSID
        string "WiFi SSID"
        default "Sweet Kevin"
        help
            SSID (network name) for the example to connect to.

    config ESP_WIFI_PASSWORD
        string "WiFi Password"
        default "02060308"
        help
            WiFi password (WPA or WPA2) for the example to use.
            
    # ... 其他 WiFi 相关配置 ...
    
endmenu
```

### 6.2 项目依赖 (idf_component.yml)

项目依赖以下组件：

```yaml
dependencies:
  espressif/esp_new_jpeg: "^0.5.0"
  idf: ">=4.4"
  lvgl/lvgl: "~8.3.0"
  esp_lcd_touch_cst816s: "^1.0"
  esp_io_expander_tca9554: "^2.0.0"
```

## 7. 编译和烧录

1. 配置项目：
```bash
idf.py menuconfig
```

2. 编译项目：
```bash
idf.py build
```

3. 烧录到开发板：
```bash
idf.py -p COM端口 flash
```

4. 监视串口输出：
```bash
idf.py -p COM端口 monitor
```

## 8. 运行流程

1. 开发板上电后，首先初始化 NVS 存储
2. 连接到配置的 WiFi 网络
3. 从网络下载指定的 JPG 图片
4. 初始化 LVGL 图形库和 LCD 显示屏
5. 解码下载的 JPG 图片为 RGB565 格式
6. 在 LCD 上显示解码后的图片

## 9. 常见问题

1. **WiFi 连接失败**：检查 SSID 和密码是否正确，确保路由器工作正常
2. **图片下载失败**：检查网络连接和图片 URL 是否有效
3. **显示异常**：检查 LCD 连接和配置参数是否正确
4. **内存不足**：减小图片尺寸或优化内存分配

## 10. 扩展功能

1. 添加按钮控制，可以下载不同的图片
2. 实现图片缓存功能，避免重复下载
3. 添加图片浏览和缩放功能
4. 实现图片轮播效果

## 11. 总结

本教程介绍了如何使用 ESP32-S3 KSDIY Korvo 开发板实现 WiFi 连接、JPG 图片下载和显示功能。通过这个项目，你可以学习到 ESP-IDF 框架、LVGL 图形库、WiFi 编程和图像处理的基础知识。希望这个教程能帮助你开始嵌入式系统开发的学习之旅！

