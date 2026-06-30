# ESP-IDF 工程结构详解（对比 Keil）

> **文档类型**：ESP-IDF **通用**入门，不绑定单一开发板。  
> **视频系列**：[A03](./文档总目录与视频教程大纲.md#三系列-a--零基础入门必拍) · **总索引**：[文档总目录与视频教程大纲](./文档总目录与视频教程大纲.md)  
> **框架全景**见 [ESP-IDF 框架整体构成](./通用-ESP-IDF框架构成.md)  
> **延伸**：[Kconfig](./通用-kconfig.md) · [menuconfig](./通用-menuconfig.md) · [分区表](./通用-partition.md) · [创建组件](./通用-创建组件.md)

如果你以前用 **Keil MDK** 写 STM32，第一次打开 ESP-IDF 工程会觉得「文件多、没有 `.uvprojx`、编译命令在终端里」。本文用 **Keil 对照** 的方式，把 **一个最小 ESP-IDF 工程里每个常见文件是干什么的** 讲清楚。

---

## 1. 一句话对照：Keil 工程 vs ESP-IDF 工程

| 概念 | Keil MDK（STM32 常见） | ESP-IDF（ESP32 系列） |
|------|------------------------|------------------------|
| 工程描述文件 | `.uvprojx` / `.uvoptx` | **顶层 + 各目录 `CMakeLists.txt`** |
| 点按钮编译 | Keil **Build** (F7) | 终端 **`idf.py build`** |
| 下载程序 | Keil **Download** + J-Link/ST-Link | **`idf.py flash`**（经 USB 串口 bootloader） |
| 看打印 | Keil **Serial Window** / 外部串口助手 | **`idf.py monitor`**（或 build 后 `-m`） |
| 芯片型号选择 | Options → Device 选 STM32xxx | **`idf.py set-target esp32s3`** 等 |
| 宏/开关配置 | 工程 C/C++ Define、或手写 `config.h` | **`menuconfig`** → 生成 **`sdkconfig`** |
| 启动文件 | `startup_stm32xxx.s` | IDF 内部 **`esp-idf/components`** 提供，一般不改 |
| 链接脚本 | `.sct` / scatter file | **`build/` 里自动生成** `sections.ld` |
| 标准库/驱动 | CMSIS + HAL 勾选进工程 | **组件（component）**：Wi-Fi、GPIO、FreeRTOS 等按需链接 |
| 用户代码入口 | `main()` | **`app_main()`**（FreeRTOS 启动后调用） |
| 输出文件 | `.axf` / `.hex` | **`build/*.bin`**、**`bootloader.bin`**、分区表 bin |
| 工程路径 | 常无硬性限制 | **路径不要含中文** |

ESP-IDF 不是「一个 IDE 工程文件包打天下」，而是：**CMake 描述怎么编 + Kconfig 描述编什么 + idf.py 把编译/烧录/串口串起来**。

---

## 2. 上电后程序怎么跑（建立直觉）

```text
Flash 里：
  bootloader（二级引导） → 读分区表 → 加载 app 分区
       ↓
  app（你的业务 + IDF 库）启动 FreeRTOS
       ↓
  调用你写的 app_main()
```

Keil 里你往往只关心 `main()`；ESP-IDF 里 **`main()` 在 IDF 内部**，应用层统一写 **`void app_main(void)`**。

---

## 3. 最小工程目录长什么样

以酷世 DIY 例程包里的 **`01.basic.hello_world`** 为例（Korvo / P4C5 都有同名工程，结构相同）：

```text
01.basic.hello_world/
├── CMakeLists.txt              ← 工程「总 Makefile」，必须
├── sdkconfig                   ← 当前工程配置（menuconfig 结果，可 git 忽略）
├── sdkconfig.defaults          ← 默认配置（随工程分发，推荐提交）
├── sdkconfig.ci                ← CI 自动化用，初学者可忽略
├── partitions.csv              ← 部分工程有：自定义 Flash 分区
├── main/                       ← 应用组件（几乎必有）
│   ├── CMakeLists.txt          ← 本组件源文件列表
│   ├── hello_world_main.c      ← app_main() 在这里
│   ├── Kconfig.projbuild       ← 部分工程有：menuconfig 菜单项
│   └── idf_component.yml       ← 部分工程有：在线组件依赖
├── components/                 ← 部分工程有：本地自定义组件
│   └── my_driver/
│       ├── CMakeLists.txt
│       ├── include/
│       └── my_driver.c
├── build/                      ← 编译输出（自动生成，勿手改、勿提交）
│   ├── hello_world.bin         ← 要烧进 Flash 的 app
│   ├── bootloader/bootloader.bin
│   ├── partition_table/partition-table.bin
│   └── ...
└── managed_components/         ← 在线组件下载缓存（首次 build 生成）
```

下面逐个说明 **你作为初学者会碰到的文件**。

---

## 4. 顶层 `CMakeLists.txt` — 相当于 Keil 的「工程根」

典型内容只有 **三行模板**（顺序不能乱）：

```cmake
cmake_minimum_required(VERSION 3.16)

include($ENV{IDF_PATH}/tools/cmake/project.cmake)
project(hello_world)
```

| 行 | 作用 |
|----|------|
| `cmake_minimum_required` | 要求 CMake 版本 |
| `include(... project.cmake)` | 接入 ESP-IDF 构建系统（找工具链、组件、链接规则） |
| `project(hello_world)` | 工程名；影响 `build/` 里输出 bin 的默认名 |

**和 Keil 对比**：Keil 在 `.uvprojx` 里点选源文件、芯片、宏；ESP-IDF 把「芯片 target、组件依赖」 largely 交给 **`idf.py set-target`** 和 **各组件 CMakeLists**，顶层文件通常 **不用改**。

---

## 5. `main/CMakeLists.txt` — 相当于 Keil 里「这个 Group 下有哪些 .c」

```cmake
idf_component_register(
    SRCS "hello_world_main.c"
    INCLUDE_DIRS "."
    REQUIRES esp_driver_gpio nvs_flash
)
```

| 参数 | 含义 |
|------|------|
| `SRCS` | 要编译的 `.c` / `.cpp` 文件（可多个） |
| `INCLUDE_DIRS` | 头文件搜索路径 |
| `REQUIRES` | **依赖哪些组件**（类似 Keil 里勾选 HAL 库；不声明可能链接报错） |
| `PRIV_REQUIRES` | 仅本组件内部使用的依赖（不向外暴露头文件） |
| `EMBED_FILES` | 把文件打进 Flash（如 HTML、证书） |

**新增一个 `.c` 文件时**：把文件名加进 `SRCS`，然后 `idf.py build`。  
**用了新的 IDF API（如 `esp_wifi.h`）**：通常要在 `REQUIRES` 里加对应组件名（如 `esp_wifi`）。

---

## 6. `main/*.c` — 你的业务代码

入口函数固定为：

```c
void app_main(void)
{
    // 你的初始化与逻辑
}
```

| 和 Keil 习惯 | 说明 |
|--------------|------|
| 不要写 `while(1)` 占死 CPU | 用 **FreeRTOS 任务**、**事件回调**、**定时器** |
| 延时 | `vTaskDelay()`，不是 `HAL_Delay()` |
| 打印 | `printf` / `ESP_LOGI("TAG", "...")` |
| 头文件里的宏 | 很多来自 **`sdkconfig.h`**（由 menuconfig 生成） |

---

## 7. `sdkconfig` / `sdkconfig.defaults` — 相当于 Keil 的 Options + 全局 `#define`

| 文件 | 谁改 | 作用 |
|------|------|------|
| **`sdkconfig`** | `idf.py menuconfig` 保存后自动生成/更新 | **当前工程**全部 CONFIG 开关（Wi-Fi、Flash 大小、日志级别…） |
| **`sdkconfig.defaults`** | 作者手写或 `idf.py save-defconfig` | 工程 **出厂默认**；首次配置或删 sdkconfig 时会合并进来 |
| **`sdkconfig.old`** | 工具自动 | menuconfig 修改前的备份 |
| **`sdkconfig.ci`** | 自动化测试 | 本地学习可忽略 |

代码里用 `CONFIG_XXX` 宏（例如 `CONFIG_IDF_TARGET` 是芯片名），来自 **`sdkconfig` → 生成 `build/config/sdkconfig.h`**。

**Keil 对照**：Options for Target → C/C++ → Define 里写的一堆宏，在 IDF 里变成 **图形化 menuconfig**，且 **按工程独立**（换目录就是另一套 sdkconfig）。

详见 [menuconfig 教程](./通用-menuconfig.md)、[Kconfig 教程](./通用-kconfig.md)。

---

## 8. `partitions.csv` — Flash「分区地图」（Keil 里没有直接对应物）

定义 Flash 里 **bootloader / app / NVS / SPIFFS / OTA** 各占哪一段。  
例程 OTA、SPIFFS、相册等会自定义；简单 hello_world 可能用 IDF 默认分区。

改过分区后常需 **`idf.py erase-flash`** 再烧录。详见 [分区表教程](./通用-partition.md)。

---

## 9. `Kconfig.projbuild` — 给 menuconfig 加「本工程专用菜单」

放在 `main/` 下，定义 SSID、URL、功能开关等 **仅本例程用的配置项**。  
编译后变成 `CONFIG_EXAMPLE_*` 宏供 C 代码使用。

酷世 wifi、mqtt、ota 等例程里 **Example Configuration** 菜单多来自此文件。

---

## 10. `idf_component.yml` — 在线组件依赖（类似 Keil 的 Pack，但是 YAML）

```yaml
dependencies:
  lvgl/lvgl: "^9.4.0"
  kevincoooool/ksdiy_p4c5_bsp: "^1.0.5"
```

| 说明 |
|------|
| 首次 **`idf.py build`** 会从 [乐鑫组件库](https://components.espressif.com) 下载到 **`managed_components/`** |
| 需 **联网** |
| P4C5 BSP 等见 [P4C5在线组件使用说明](./P4C5在线组件使用说明.md) |

Keil 里你手动拷贝库文件夹；IDF 用 **组件管理器** 自动拉版本。

---

## 11. `components/` — 本地自定义组件

目录结构：

```text
components/my_bsp/
  CMakeLists.txt
  include/my_bsp.h
  my_bsp.c
```

**何时用**：多个例程共用的驱动/UI，不想每次复制粘贴。  
顶层 `CMakeLists.txt` 一般 **不用** 写 `add_subdirectory`；IDF 会 **自动扫描** 工程下的 `components/`。

详见 [创建组件](./通用-创建组件.md)。

---

## 12. `build/` — 编译产物（别当源码改）

| 输出 | 用途 |
|------|------|
| `xxx.bin` | 应用程序，烧录到 app 分区 |
| `bootloader/bootloader.bin` | 引导程序 |
| `partition_table/partition-table.bin` | 分区表 |
| `xxx.elf` / `xxx.map` | 调试、查符号/内存占用 |
| `compile_commands.json` | clangd/IDE 代码跳转 |

**Keil 对照**：类似 `Objects/`、`Listings/` 里的 axf/hex，但 ESP 常烧 **bin + 多镜像**。

切换芯片、大改 menuconfig、莫名链接错误时：**删掉整个 `build/` 再 `idf.py build`**（相当于 Keil 里 Clean + Rebuild）。

---

## 13. `idf.py` 常用命令 ↔ Keil 操作

| 目的 | ESP-IDF | Keil 近似 |
|------|---------|-----------|
| 选芯片 | `idf.py set-target esp32s3` | Select Device |
| 编译 | `idf.py build` | Build (F7) |
| 烧录 | `idf.py flash` | Download |
| 串口 log | `idf.py monitor` | UART 窗口 |
| 一条龙 | `idf.py build flash monitor` | Build + Download + 看串口 |
| 图形配置 | `idf.py menuconfig` | Options 对话框 |
| 清配置 | 删 `sdkconfig` 再 build | 恢复默认 Define |
| 全片擦除 | `idf.py erase-flash` | 擦除整片 Flash |

指定串口：`idf.py -p COM15 flash monitor`

---

## 14. 酷世例程包里的「多工程」结构

百度网盘解压后：

```text
Korvo_Firmware/          或    P4_C5_4.3_Firmware/
  01.basic.hello_world/        ← 每个子文件夹 = 独立 ESP-IDF 工程
  02.beginner.wifi_station/    ← 各有自己的 CMakeLists、sdkconfig、build/
  components/                  ← 多个例程共用的本地组件
```

**重要**：从 `hello_world` 换到 `wifi_station`，必须 **`cd` 到新目录** 再编译，不能共用一个 `build/`。  
这和 Keil 里 **打开另一个 `.uvprojx`** 是一类操作。

---

## 15. 推荐阅读顺序（第一次读工程）

1. **`main/*.c`** 找 `app_main()` — 业务从哪开始  
2. **`main/CMakeLists.txt`** — 有哪些源文件、依赖谁  
3. **`main/Kconfig.projbuild`**（若有）— menuconfig 要配什么  
4. **顶层 `CMakeLists.txt`** — 通常只看工程名  
5. **`sdkconfig.defaults`** — 作者预设了哪些选项  
6. **`idf_component.yml`**（若有）— 在线拉了什么库  
7. **`components/`**（若有）— BSP/显示等从哪封装  

然后再去看 [GPIO](./通用-gpio.md)、[Wi-Fi](./通用-wifi.md) 等 API 教程。

---

## 16. 常见问题

| 现象 | 可能原因 |
|------|----------|
| 找不到 `app_main` 重复定义 | 多个 `.c` 都写了 `app_main` |
| `undefined reference to xxx` | `main/CMakeLists.txt` 缺 `REQUIRES 组件名` |
| 改了 menuconfig 代码里宏不变 | 未保存 menuconfig；或未 `#include "sdkconfig.h"` |
| 换例程编译怪错 | 还在旧目录的 `build/` 里编；应 `cd` 新工程 |
| 中文路径编译失败 | 工程路径移到纯英文目录 |

---

## 17. 下一步

- 动手跑通 [例程开发流程](./例程开发流程与公共组件.md) 里的 hello_world / board_test  
- Korvo：[Korvo例程学习指南](./Korvo例程学习指南.md)  
- P4C5：[P4C5例程学习指南](./P4C5例程学习指南.md)  

酷世DIY · Kevincoooool
