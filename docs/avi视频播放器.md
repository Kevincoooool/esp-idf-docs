


          
# ESP32-S3 AVI播放器教程

## 项目概述

本教程将介绍如何使用ESP32-S3开发板实现AVI视频播放功能。该项目基于ESP-IDF框架，使用了LVGL图形库、ESP-AVI播放器组件和ESP-JPEG解码器等组件，实现了从SPIFFS文件系统读取AVI文件并在LCD屏幕上播放的功能，同时通过ES8311音频编解码器输出声音。

## 硬件要求

- ESP32-S3开发板（KSDIY Korvo开发板）
- LCD显示屏（ST7789驱动，分辨率280x240）
- ES8311音频编解码器
- 扬声器

## 软件依赖

项目依赖以下组件（在idf_component.yml中定义）：
- espressif/avi_player: "^1.0.0"
- espressif/esp_new_jpeg: "^0.5.0"
- lvgl/lvgl: "~8.3.0"
- esp_lcd_touch_cst816s: "^1.0"
- esp_io_expander_tca9554: "^2.0.0"
- espressif/esp_codec_dev: "^1.3.1"

## 项目结构

项目主要包含以下文件：
1. `app_main.c` - 主程序入口，包含AVI播放器初始化和播放逻辑
2. `app_speech.c/h` - 音频编解码器初始化和I2S配置
3. `ksdiy_lcd_port.c/h` - LCD显示屏初始化和配置
4. `CMakeLists.txt` - 项目构建配置
5. `idf_component.yml` - 组件依赖配置

## 详细实现步骤

### 1. 初始化系统组件

首先，我们需要初始化NVS（非易失性存储）和SPIFFS文件系统：

```c
/* 初始化NVS */
esp_err_t err = nvs_flash_init();
if (err == ESP_ERR_NVS_NO_FREE_PAGES || err == ESP_ERR_NVS_NEW_VERSION_FOUND) {
    ESP_ERROR_CHECK(nvs_flash_erase());
    err = nvs_flash_init();
}
ESP_ERROR_CHECK(err);

/* 初始化SPIFFS文件系统 */
ESP_LOGI(TAG, "Initializing SPIFFS");
esp_vfs_spiffs_conf_t conf = {
    .base_path = "/spiffs",
    .partition_label = "storage",
    .max_files = 2,
    .format_if_mount_failed = true
};
esp_err_t ret = esp_vfs_spiffs_register(&conf);
```

### 2. 初始化音频编解码器

音频编解码器初始化在`app_speech.c`中实现，主要包括I2C、I2S和ES8311编解码器的初始化：

```c
/* 初始化音频编解码器 */
void Codec_I2S_init(void) {
    // 初始化I2C总线
    InitializeCodecI2c();
    // 创建I2S通道
    CreateDuplexChannels(AUDIO_I2S_GPIO_MCLK, AUDIO_I2S_GPIO_BCLK, AUDIO_I2S_GPIO_WS, 
                         AUDIO_I2S_GPIO_DOUT, AUDIO_I2S_GPIO_DIN);
    // 初始化编解码器
    Init_codec();
    // 配置功放使能引脚
    esp_rom_gpio_pad_select_gpio(PA_GPIO_NUM);
    gpio_set_direction(PA_GPIO_NUM, GPIO_MODE_OUTPUT);
    gpio_set_level(PA_GPIO_NUM, 1);
}
```

### 3. 初始化LCD显示屏

LCD显示屏初始化在`ksdiy_lcd_port.c`中实现：

```c
void ksdiy_lvgl_lcd_init(void) {
    // 初始化SPI总线
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
    
    // 配置LCD面板
    ESP_ERROR_CHECK(esp_lcd_new_panel_st7789(io_handle, &panel_config, &panel_handle));
    ESP_ERROR_CHECK(esp_lcd_panel_reset(panel_handle));
    ESP_ERROR_CHECK(esp_lcd_panel_init(panel_handle));
    ESP_ERROR_CHECK(esp_lcd_panel_swap_xy(panel_handle, true));
    ESP_ERROR_CHECK(esp_lcd_panel_mirror(panel_handle, false, true));
    ESP_ERROR_CHECK(esp_lcd_panel_invert_color(panel_handle, true));
    ESP_ERROR_CHECK(esp_lcd_panel_disp_on_off(panel_handle, true));
}
```

### 4. JPEG解码函数

为了在LCD上显示视频帧，我们需要将JPEG格式的视频帧解码为RGB565格式：

```c
jpeg_error_t esp_jpeg_decode_one_picture(uint8_t *input_buf, int len, uint8_t **output_buf, int *out_len) {
    // 创建JPEG解码器配置
    jpeg_dec_config_t config = DEFAULT_JPEG_DEC_CONFIG();
    config.output_type = j_type;  // RGB565_BE格式
    config.rotate = j_rotation;   // 不旋转
    
    // 创建JPEG解码器句柄
    jpeg_dec_handle_t jpeg_dec = NULL;
    ret = jpeg_dec_open(&config, &jpeg_dec);
    
    // 解析JPEG头部信息
    ret = jpeg_dec_parse_header(jpeg_dec, jpeg_io, out_info);
    rgb_width = out_info->width;
    rgb_height = out_info->height;
    
    // 分配输出缓冲区
    *out_len = out_info->width * out_info->height * 2;  // RGB565格式，每像素2字节
    out_buf = jpeg_calloc_align(*out_len, 16);
    jpeg_io->outbuf = out_buf;
    *output_buf = out_buf;
    
    // 开始解码JPEG
    ret = jpeg_dec_process(jpeg_dec, jpeg_io);
    
    // 释放资源
    jpeg_dec_close(jpeg_dec);
    free(jpeg_io);
    free(out_info);
    
    return ret;
}
```

### 5. AVI播放器回调函数

AVI播放器需要三个主要回调函数：视频帧回调、音频帧回调和播放结束回调：

```c
// 视频帧回调函数
void video_write(frame_data_t *data, void *arg) {
    // 解码JPEG视频帧
    esp_jpeg_decode_one_picture(data->data, data->data_bytes, &img_rgb565, &Rgbsize);
    // 在LCD上显示解码后的帧
    esp_lcd_panel_draw_bitmap(panel_handle, 20, 0, rgb_width + 20, rgb_height, img_rgb565);
    // 释放解码后的缓冲区
    free(img_rgb565);
}

// 音频帧回调函数
void audio_write(frame_data_t *data, void *arg) {
    // 通过编解码器播放音频
    esp_codec_dev_write(output_dev_, data->data, data->data_bytes);
}

// 播放结束回调函数
void avi_play_end(void *arg) {
    ESP_LOGI(TAG, "Play end");
    end_play = true;
    cnt++;
    if(cnt > 107)
        cnt = 94;
    // 播放下一个视频文件
    sprintf(file_name, "/spiffs/h0%03d.avi", cnt);
    avi_player_play_from_file((char *)file_name);
}
```

### 6. 初始化并启动AVI播放器

最后，我们初始化AVI播放器并开始播放视频：

```c
// 初始化AVI播放器
avi_player_config_t config = {
    .buffer_size = 50 * 1024,
    .audio_cb = audio_write,
    .video_cb = video_write,
    .avi_play_end_cb = avi_play_end,
    .coreID = 1,
};

avi_player_init(config);

// 开始播放第一个视频文件
avi_player_play_from_file("/spiffs/h0094.avi");
```

## 视频文件准备

项目中的注释提供了使用FFmpeg转换视频文件的命令：

```bash
ffmpeg -i .\badapple.avi -t 30 -vcodec mjpeg -vf scale=280:240 -r 14 -q:v 60 -acodec pcm_s16le -ar 16000 output.avi
```

参数说明：
- `-t` 是视频时长（秒）
- `-r` 是帧率
- `-q:v` 是JPEG质量，数值越大输出文件越小，数值越小越清晰
- `-ar` 是音频采样率

## 总结

本项目实现了在ESP32-S3开发板上播放AVI视频的功能，包括视频解码和显示、音频播放等。通过使用ESP-IDF提供的组件和库，我们可以轻松实现多媒体播放功能。

要运行此项目，您需要：
1. 准备好符合要求的AVI视频文件
2. 将视频文件放入SPIFFS分区
3. 编译并烧录程序到ESP32-S3开发板
4. 连接LCD显示屏和扬声器

项目会自动播放SPIFFS中的视频文件，并在一个视频播放完成后自动播放下一个视频。

        