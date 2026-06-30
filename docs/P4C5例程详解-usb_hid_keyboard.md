# P4C5 · USB HID 键盘

> **适用开发板**：P4C5 · **工程** `02.beginner.usb_hid_keyboard` · **target** `esp32p4`  
> **连接**：Type-C 接 **PC**（Device 模式），板子被识别为 USB 键盘  

演示 **CherryUSB** 实现 HID 键盘：向 PC 发送按键报告。MIPI 屏显示 USB 初始化状态。配对学习：`02.beginner.usb_hid_mouse`（报告格式不同）。

---

## 1. 这个例子在干嘛

1. 注册 **HID 键盘** 设备（报告描述符告诉 PC「我是键盘」）  
2. 启动后 **自动发送一次按键 A**（光标在记事本会看到 `a`）  
3. 之后空闲，不再发键（可改代码循环发送）  

---

## 2. 使用前注意

- 用 **数据线** 连接 PC USB（非仅充电线）  
- 烧录前若光标在文本框，自动打 `a` 可能干扰，先切到安全窗口  
- 设备管理器 → 键盘 → 多 **HID Keyboard Device**（名含 CherryUSB HID DEMO）  

---

## 3. 编译烧录

```powershell
cd P4_C5_4.3_Firmware\02.beginner.usb_hid_keyboard
idf.py set-target esp32p4
idf.py build flash monitor
```

---

## 4. 运行现象

**屏幕**：`USB device init` → `USB configured` → `sending key demo`  

**PC**：枚举键盘 → 输入一个 `a`  

---

## 5. 关键文件

| 文件 | 作用 |
|------|------|
| `main/app_main.c` | 屏显 + `hid_keyboard_init1` + 测试按键 |
| `main/usbx_demo.c` | 报告描述符、`hid_keyboard_test` |
| `main/usb_config.h` | CherryUSB 配置 |

---

## 6. HID 键盘报告（8 字节）

```text
[修饰键, 保留, 键1, 键2, 键3, 键4, 键5, 键6]
```

- 修饰键：`0x02` = Left Shift → Shift+A = 大写 `A`  
- 键码：`HID_KBD_USAGE_A` 等，见 USB HID Usage Tables  

`hid_keyboard_report_desc`（63 字节）是 **Report Descriptor**，定义键位布局。

发送流程：`hid_keyboard_test` 填报告 → **中断 IN 端点** → PC。`hid_state`（IDLE/BUSY）防止重入。

---

## 7. 与 HID 鼠标的区别

| | 键盘 | 鼠标 |
|--|------|------|
| 报告长度 | 8 字节按键 | 通常 4 字节 按钮+XY |
| 描述符 | 按键矩阵 | 指针设备 |

见 `02.beginner.usb_hid_mouse`。

---

## 8. 常见问题

| 现象 | 处理 |
|------|------|
| PC 无设备 | 线材；USB Device 模式是否启用；驱动 |
| 有设备无按键 | 是否调用了 test；端点是否 BUSY |
| 乱码 | 键码/修饰键字节错误 |

---

## 9. 动手改造

1. 改 `HID_KBD_USAGE_A` 为空格 `0x2C`  
2. `while(1)` 里每 2 s 发一个字符  
3. 触摸屏虚拟键盘映射到 HID 报告  
4. `04.advanced.usb_extended_display` — 复合 USB 应用  

Korvo 若有同类 HID 例程见 [Korvo usb_hid](./Korvo例程详解-usb_hid_keyboard.md)。

酷世DIY · Kevincoooool
