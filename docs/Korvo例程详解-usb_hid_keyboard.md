# Korvo · USB HID 键盘

> **适用开发板**：Korvo S3 · **工程** `02.beginner.usb_hid_keyboard` · **target** `esp32s3`  
> **连接**：USB 接 PC，枚举为 **HID 键盘**

与 P4C5 同名工程结构类似：`usbx_demo.c` 里 HID 报告描述符 + 发送 8 字节键盘报告。Korvo **无 MIPI 大 UI**，主要靠串口 log。

P4 带屏说明见 [P4C5 usb_hid_keyboard](./P4C5例程详解-usb_hid_keyboard.md)。

---

## 1. 这个例子在干嘛

1. 注册 USB **Device** 类 HID 键盘  
2. 示例发送一次或若干次按键报告（如 `'A'`）  
3. PC 记事本可见字符输入  

---

## 2. 编译烧录

```powershell
cd Korvo_Firmware\02.beginner.usb_hid_keyboard
idf.py set-target esp32s3
idf.py build flash monitor
```

用 **数据线** 连接 Korvo USB 口与 PC。烧录前光标勿在重要文本框。

---

## 3. 关键文件

| 文件 | 作用 |
|------|------|
| `main/app_main.c` | init + 测试 |
| `main/usbx_demo.c` | 描述符、`hid_keyboard_test` |

---

## 4. HID 报告（8 字节）

```text
[修饰键, 保留, 键1..键6]
```

- 修饰键 `0x02` = Shift → 大写  
- 键码见 `HID_KBD_USAGE_*`  

**Report Descriptor** 定义设备是键盘；**中断 IN** 端点送报告。

---

## 5. 与鼠标 HID

`02.beginner.usb_hid_mouse` 报告结构不同（按钮+XY），对比学习。

---

## 6. 常见问题

| 现象 | 处理 |
|------|------|
| PC 无设备 | 线材；USB OTG/Device 配置 |
| 无按键 | 是否调用 test；端点 busy |

---

## 7. 动手改造

周期发送字符；触摸屏映射键码（若有屏工程）。

酷世DIY · Kevincoooool
