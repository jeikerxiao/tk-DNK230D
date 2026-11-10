# 正点原子 DNK230D（K230D BOX）CanMV 综合例程

> 本目录是 DNK230D 开发板的 **综合例程（CanMV 版）**：一个用 **LVGL 图形库** 打造的"类智能手机桌面系统"。开机后有 **锁屏页 → 上滑进入桌面 → 点击图标打开各种 APP**，内置计算器、时钟、相册、设置、自由画板、硬件测试等应用，还有一个 **AI Hub（AI 体验中心）**，可以一键重启运行 19 个人脸 / 人体 / 手势 / 物体 / 语音等 AI 演示。
>
> 它把[标准例程](../标准例程-CanMV版/README.md)里学到的零散知识点（屏幕、触摸、PWM、RTC、摄像头、KPU AI）**整合成一个完整、可交互、好看的产品级工程**，是学完基础例程后的"毕业设计"式范例。难度：**中高级**。

---

## 目录

- [1. 例程简介](#1-例程简介)
- [2. 硬件介绍与接线](#2-硬件介绍与接线)
- [3. 相关前置知识](#3-相关前置知识)
- [4. 工程目录结构](#4-工程目录结构)
- [5. 系统架构与代码逻辑](#5-系统架构与代码逻辑)
- [6. 内置 APP 一览](#6-内置-app-一览)
- [7. AI Hub 演示索引](#7-ai-hub-演示索引)
- [8. 部署与运行](#8-部署与运行)
- [9. 实验效果](#9-实验效果)
- [10. 注意事项](#10-注意事项)
- [11. v1.2 与 v1.3 的区别](#11-v12-与-v13-的区别)
- [12. 常见问题 FAQ](#12-常见问题-faq)
- [13. 总结](#13-总结)

---

## 1. 例程简介

**这个综合例程到底是个什么东西？**

简单说，它让你的 K230D BOX 开发板看起来、用起来像一台**小手机 / 智能面板**：

- 上电后先显示 **锁屏界面**（壁纸 + 超大时钟 + "Swipe up to open" 提示），手指**向上滑动**解锁；
- 进入 **桌面（Home Screen）**：顶部状态栏显示时间，中间是可左右翻页的应用图标网格，底部是常驻 Dock 栏（Settings / Clock / Calculator / AI Hub）；
- 点击任意图标，对应的 **APP 以动画方式打开**，可以触摸操作，按物理按键 KEY0 可返回锁屏；
- 打开 **AI Hub**，里面列出了 19 个 AI 演示（人脸检测、手势识别、物体检测、关键词唤醒……），点 "Reboot to Run" 确认后，开发板会把该演示脚本设为开机脚本并**重启运行**，直接进入摄像头 AI 画面。

**它和标准例程的关系**

| 对比项 | 标准例程 | 综合例程（本目录） |
| --- | --- | --- |
| 形态 | 一个实验一个 `main.py`，各自独立 | 一个完整的图形系统工程 |
| 界面 | 大多是命令行 / 简单画面 | LVGL 精美图形桌面，可触摸交互 |
| 目的 | 学**单个知识点** | 学**如何把知识点组织成大工程** |
| AI 演示 | 每个实验独立运行 | 由 AI Hub 统一入口，重启切换 |

> [!NOTE]
> **为什么要"重启运行"AI 演示？** 因为摄像头 + KPU AI 推理需要独占底层媒体资源，和桌面 GUI（LVGL 占用屏幕做刷新）没法在同一时刻和平共存。所以本系统采用"把要跑的 AI 脚本复制成开机脚本 `/sdcard/main.py`，再 `machine.reset()` 重启"的方式，干净地切换到 AI 演示。

**适合谁？**

- 已经跟完[标准例程](../标准例程-CanMV版/README.md)、想看看"工业级 / 产品级代码长什么样"的同学；
- 想学习 **LVGL 图形界面开发**（事件、动画、布局、自定义控件）的嵌入式爱好者；
- 做课程设计 / 毕设，需要一个"带屏幕交互 + AI"完整 demo 打样的同学。

---

## 2. 硬件介绍与接线

### 2.1 本例程用到的板载资源

综合例程**全部使用板载资源，无需外接任何杜邦线 / 模块**。`main()` 函数在启动时统一初始化了下列硬件（见 `main.py` 末尾）：

| 硬件对象 | 引脚 / 通道 | 作用 | 在哪个 APP 用到 |
| --- | --- | --- | --- |
| **LCD 显示屏** | MIPI DSI，640×480，ST7701 | 整个 GUI 的显示载体 | 全部 |
| **背光 BL** | GPIO5（`bl_valid=1`，高电平点亮） | 控制屏幕背光开关 | 全部 |
| **触摸屏** | MIPI / I2C 电容触摸 | 点击、滑动、绘图输入 | 全部 |
| **红色 LED（LEDR）** | GPIO61，**PWM 通道 1** | 可调亮度的红灯 | Tester（硬件测试） |
| **蓝色 LED（LEDB）** | GPIO59，**PWM 通道 5** | 可调亮度的蓝灯 | Tester（硬件测试） |
| **蜂鸣器（Buzzer）** | GPIO60，**PWM 通道 0** | 可调音量、可调频率发声 | Tester（硬件测试） |
| **按键 KEY0（Button0）** | GPIO34（低电平有效） | **按下返回锁屏** | 桌面全局 |
| **按键 KEY1（Button1）** | GPIO35（低电平有效） | 预留 | — |
| **按键 KEY2（Button2）** | GPIO0（高电平有效，兼启动模式选择） | 预留 | — |
| **RTC 实时时钟** | 芯片内部 | 提供时间，初值 2024-12-12 08:00:00 | 锁屏 / 状态栏 / 时钟 APP / 设置 APP |

> [!NOTE]
> **和标准例程一致的引脚**：红色 LED=GPIO61、蓝色 LED=GPIO59，与[实验1 跑马灯](../标准例程-CanMV版/基础实验例程/实验1%20跑马灯实验/README.md)完全对应；蜂鸣器用 PWM 发声，对应[实验2 蜂鸣器](../标准例程-CanMV版/基础实验例程/实验2%20蜂鸣器实验/README.md)和[实验6 呼吸灯](../标准例程-CanMV版/基础实验例程/实验6%20呼吸灯实验/README.md)（LED 亮度也是 PWM 调节）。综合例程只是把它们包进了 `LED` / `Buzzer` / `Button` 类里统一管理。

### 2.2 AI Hub 演示额外用到的资源

当你在 AI Hub 里运行某个 AI 演示时，演示脚本（`APP/AI Hub/*.py`）还会用到：

| 资源 | 说明 |
| --- | --- |
| **摄像头** | MIPI CSI 摄像头，采集实时画面 |
| **KPU** | K230D 的 AI 加速器，运行 `.kmodel` 神经网络模型 |
| **麦克风** | 关键词唤醒（keyword_spotting）演示用 |
| **模型文件** | 位于开发板 `/sdcard/examples/kmodel/*.kmodel` |
| **算法库** | 位于开发板 `/sdcard/libs/`（PipeLine、AIBase、AI2D 等）和 `aidemo` 内置库 |

> [!WARNING]
> **AI Hub 的模型文件不在本仓库里！** 演示脚本从 `/sdcard/examples/kmodel/` 读取 `.kmodel` 模型、从 `/sdcard/examples/utils/` 读取辅助数据。这些文件由 **CanMV 固件 / 标准例程资源包**提供。也就是说，你的 TF 卡（或板载存储）里必须已经有 `examples/kmodel`、`examples/utils`、`libs` 这些目录，AI Hub 的演示才能正常跑起来。若缺失，运行演示会报"打不开 kmodel"之类的错误。

### 2.3 接线 / 硬件避坑清单

> [!WARNING]
> - 本例程**不需要你接线**，但请确保摄像头、屏幕排线已正确插好（MIPI 排线插反会烧坏，务必断电操作，详见[标准例程 README 的硬件章节](../标准例程-CanMV版/README.md#2-硬件介绍)）。
> - 供电请用 **5V/2A 及以上**。GUI + 摄像头 + KPU 同时工作电流较大，供电不足会重启、黑屏、IDE 掉线。
> - 运行 AI 演示时开发板**发热属正常**，不要遮挡散热面。

---

## 3. 相关前置知识

学这个综合例程前，最好先了解下面几个概念（不熟也没关系，边看边查）：

- **LVGL（Light and Versatile Graphics Library）**：一套开源的嵌入式图形界面库，用"对象（obj）+ 控件（label / btn / img / slider …）+ 事件回调 + 动画"的方式搭建界面。本工程的整个桌面都是 LVGL 画出来的。CanMV 固件已内置 `lvgl` 模块，`import lvgl as lv` 即可用。
- **屏幕刷新回调 `flush_cb`**：LVGL 把画好的一帧交给 `LCD.lvgl_flush_cb()`，再调用 `display.show_image()` 送到屏幕。这是"软件画界面 → 硬件显示"的桥梁。
- **触摸输入回调 `read_cb`**：`Touch.lvgl_read_cb()` 把触摸坐标喂给 LVGL，界面才能响应点击 / 滑动。
- **PWM（脉宽调制）**：通过调节占空比来"调光""调音量"。本工程 LED 亮度、蜂鸣器音量都用 PWM。
- **RTC（实时时钟）**：芯片内部时钟，掉电后靠电池 / 电容维持，提供年月日时分秒。锁屏和状态栏的时间就来自它。
- **KPU / kmodel**：K230D 的 AI 加速器和模型文件格式。AI Hub 里的每个演示本质是标准例程里的一个 AI 实验（见[第 7 节](#7-ai-hub-演示索引)）。
- **`machine.reset()`**：软件复位开发板，相当于"重启"。AI Hub 用它来切换到演示脚本。

---

## 4. 工程目录结构

本目录下按 **CanMV 固件版本**分成两个几乎相同的工程文件夹，请**按你的固件版本二选一**（区别见[第 11 节](#11-v12-与-v13-的区别)）：

```
综合例程-CanMV版/
├── README.md                     ← 本文档（综合例程总览）
├── CanMV v1.2/                   ← 适配 CanMV 固件 v1.2 的工程
│   ├── README.md                 ← v1.2 运行说明
│   ├── main.py                   ← 开机入口脚本（= CanMV Sample/main.py）
│   └── CanMV Sample/             ← 需整体拷贝到 /sdcard/ 的资源工程
│       ├── main.py               ← GUI 主程序（3500+ 行，整个桌面系统）
│       ├── APP/                  ← 各 APP 的资源与外部脚本
│       │   ├── icons/            ← 桌面应用图标（90×90 png）
│       │   ├── Settings/         ← 设置 APP 的图标资源
│       │   ├── Tester/           ← 硬件测试 APP 的音量图标资源
│       │   └── AI Hub/           ← 19 个 AI 演示脚本(.py) + 预览图 + 图标
│       ├── Photos/               ← 相册 APP 用的示例图片
│       └── Fonts/                ← LVGL 字库文件（.bin，各种字号）
└── CanMV v1.3/                   ← 适配 CanMV 固件 v1.3 的工程（结构同上）
    └── ...
```

> [!NOTE]
> - **`main.py` 有两份且内容相同**：版本根目录下的 `main.py` 和 `CanMV Sample/main.py` 是同一个文件。前者方便你直接拷成 `/sdcard/main.py` 当开机脚本，后者是资源工程里的原件。
> - **`CanMV Sample` 这个文件夹名不能改**：代码里写死了资源路径 `RESOURCES_PATH = "/sdcard/CanMV Sample/"`，改名会导致字体、图片、图标全部加载失败。

---

## 5. 系统架构与代码逻辑

整个工程的核心是 `CanMV Sample/main.py`（约 3547 行），可以拆成"**硬件驱动层 + GUI 层 + 主循环**"三部分。

### 5.1 硬件驱动层（封装成类，方便 GUI 调用）

| 类 | 作用 | 关键方法 |
| --- | --- | --- |
| `LCD` | 初始化 ST7701 屏 + 背光，提供 LVGL 刷新回调 | `lvgl_flush_cb()`、`on()`、`off()` |
| `Touch` | 初始化触摸，提供 LVGL 触摸读取回调 | `lvgl_read_cb()` |
| `LED` | 封装 LED，支持 GPIO 开关或 **PWM 调光** | `on()`、`off()`、`set_brightness(0~100)` |
| `Buzzer` | 封装蜂鸣器，PWM 控制音量和频率 | `set_loudness(0~100)`、`set_frequency(Hz)` |
| `Button` | 封装按键输入 | `is_pressing()` |
| `ClockManager` | 封装 RTC，读写时间 | `set_time(...)`、`get_time()` |

这一层把[标准例程](../标准例程-CanMV版/README.md)里"跑马灯、蜂鸣器、按键、RTC、LCD、触摸屏"等独立实验的代码，**收敛成可复用的对象**，GUI 层只需要 `ledr.set_brightness(50)` 这样调用即可。

### 5.2 GUI 层：一个大的 `GUI` 类

`GUI` 类内部用**嵌套类**组织了整个桌面的每一块界面，层级如下：

```
GUI
├── LockScreen            锁屏页
│   ├── HomeBar           底部"上滑解锁"提示条 + 滑动手势 + 动画
│   ├── ClockTime         超大号时钟（150 号字库）
│   └── Wallpaper         壁纸（读取 png 显示）
└── HomeScreen            桌面
    ├── StatusBar         顶部状态栏（时间等）
    ├── AppConv           应用图标网格（可左右翻页）
    ├── Dock              底部常驻栏（Settings/Clock/Calculator/AI Hub）
    ├── Wallpaper         桌面壁纸
    └── AppManager        应用管理器（核心）
        ├── AppBase       所有 APP 的基类（打开动画、关闭、home bar 等公共逻辑）
        ├── AppTemplate   空白 APP 模板（供你照着写新 APP）
        ├── AppAIHub      AI 体验中心
        ├── AppCalculator 计算器
        ├── AppPhotos     相册
        ├── AppSettings   设置
        ├── AppClock      时钟（本地时钟 + 秒表）
        ├── AppFreeform   自由画板
        ├── AppTester     硬件测试
        ├── AppLoader     负责"点击图标→实例化对应 APP→播放打开动画"
        └── FullScreenMSG 全屏弹窗（确认 / 提示）
```

**APP 是怎么注册和打开的？** `AppManager` 里有一张"应用字典"：

```python
self.app_dict = {
    "Template":   self.AppTemplate,
    "AI Hub":     self.AppAIHub,
    "Calculator": self.AppCalculator,
    "Photos":     self.AppPhotos,
    "Settings":   self.AppSettings,
    "Clock":      self.AppClock,
    "Freeform":   self.AppFreeform,
    "Tester":     self.AppTester,
}
```

点击图标时，`AppLoader.load_app()` 根据图标名字从字典里取出对应的类并实例化，播放"放大打开"动画。**字典里没有的图标（如桌面上的 Books / Files / Calendar / Health / Music / Weather 等）只是占位装饰，点了不会打开任何东西**——它们是留给你自己扩展的位置。

### 5.3 AI Hub 的运行机制（重点）

AI Hub 里每个演示是 `APP/AI Hub/` 下的一个**独立脚本**。点击演示 → "Reboot to Run" → 确认后，执行下面这段逻辑：

```python
def run(self, demo_script_path):
    try:
        os.remove("/sdcard/main.py")          # 删掉旧的开机脚本
    except Exception:
        pass
    with open(demo_script_path, "rb") as f:   # 读取选中的 AI 演示脚本
        code_src = f.read()
    with open("/sdcard/main.py", "wb") as f:  # 写成新的开机脚本
        f.write(code_src)
    machine.reset()                           # 重启，开机即运行该演示
```

> [!IMPORTANT]
> 这意味着：**运行 AI 演示会覆盖 `/sdcard/main.py`（也就是桌面 GUI 本身）**。想回到桌面系统，需要把 `CanMV Sample/main.py` 重新拷贝回 `/sdcard/main.py` 再重启（见[第 8 节](#8-部署与运行)）。

### 5.4 主循环

```python
while True:
    lv.task_handler()        # 驱动 LVGL：处理刷新、事件、动画
    gc.collect()             # 回收内存，防止长时间运行内存碎片
    if button0.is_pressing():# 按下 KEY0
        gui.lock()           # 回到锁屏页
```

`lv.task_handler()` 必须被高频反复调用，LVGL 才能持续刷新界面、响应触摸、播放动画。这是所有 LVGL 工程的通用主循环写法。

---

## 6. 内置 APP 一览

桌面里**真正能用**的 8 个 APP（`app_dict` 中注册的）：

| APP | 中文 | 功能 | 用到的硬件 / 知识点 |
| --- | --- | --- | --- |
| **Calculator** | 计算器 | 触摸按键做四则运算，有历史记录区 | LVGL 按钮 / 布局 |
| **Clock** | 时钟 | 本地表盘时钟（随 RTC 走）+ 秒表（Start/Stop/Reset） | LVGL 动画、定时器、RTC |
| **Photos** | 相册 | 浏览 `Photos/` 目录里的示例图片 | LVGL 图片控件、文件读取 |
| **Settings** | 设置 | Profile（显示"正点原子 / ALIENTEK"）+ Date & Time 日期时间设置，写入 RTC | RTC、LVGL 滚轮 |
| **Freeform** | 自由画板 | 手指在 canvas 上涂鸦，多色画笔，可 Clean 清屏 | 触摸坐标、LVGL canvas |
| **Tester** | 硬件测试 | 调 LEDR/LEDB 亮度、蜂鸣器音量与频率（0~4000Hz） | **PWM、GPIO**（对应跑马灯/呼吸灯/蜂鸣器实验） |
| **AI Hub** | AI 体验中心 | 19 个 AI 演示的统一入口，重启运行 | 摄像头、KPU、`machine.reset()` |
| **Template** | APP 模板 | 一个空白按钮，是给开发者**照着写新 APP 的骨架** | LVGL APP 开发范式 |

> [!TIP]
> **想自己加一个 APP？** 复制 `AppTemplate` 类改名，在里面用 LVGL 搭你的界面，然后把它加进 `app_dict`，再在 `main()` 里用 `app_conv.add_app(页码, 图标路径, "名字", 颜色)` 添加一个图标即可。这就是 `Template` 存在的意义。

---

## 7. AI Hub 演示索引

AI Hub 共列出 **19 个 AI 演示 + 1 个"More"占位**。每个演示都对应[标准例程 · AI 类实验](../标准例程-CanMV版/AI类实验例程)里的一个实验——**想深入了解某个演示的原理、模型、参数，直接点下表链接看标准例程里那份保姆级教程即可**（本综合例程不再重复讲解每个 AI 算法）。

| # | AI Hub 名称 | 脚本文件 | 功能 | 对应标准例程详解 |
| :-: | --- | --- | --- | --- |
| 1 | Face Detection | `face_detection.py` | 人脸检测 | [实验1 人脸检测](../标准例程-CanMV版/AI类实验例程/实验1%20人脸检测实验/README.md) |
| 2 | Face Landmark | `face_landmark.py` | 人脸关键点检测 | [实验2 人脸关键点检测](../标准例程-CanMV版/AI类实验例程/实验2%20人脸关键点检测实验/README.md) |
| 3 | Face Pose | `face_pose.py` | 人脸姿态（俯仰/偏航/翻滚） | [实验3 人脸姿态检测](../标准例程-CanMV版/AI类实验例程/实验3%20人脸姿态检测实验/README.md) |
| 4 | Person Detection | `person_detection.py` | 人体检测 | [实验5 人体检测](../标准例程-CanMV版/AI类实验例程/实验5%20人体检测实验/README.md) |
| 5 | Person Keypoint | `person_keypoint_detect.py` | 人体骨骼关键点 | [实验6 人体关键点检测](../标准例程-CanMV版/AI类实验例程/实验6%20人体关键点检测实验/README.md) |
| 6 | Falldown Detect | `falldown_detect.py` | 跌倒检测 | [实验7 跌倒检测](../标准例程-CanMV版/AI类实验例程/实验7%20跌倒检测实验/README.md) |
| 7 | Hand Detection | `hand_detection.py` | 手掌检测 | [实验8 手掌检测](../标准例程-CanMV版/AI类实验例程/实验8%20手掌检测实验/README.md) |
| 8 | Hand Recognition | `hand_recognition.py` | 静态手势识别 | [实验9 手势识别](../标准例程-CanMV版/AI类实验例程/实验9%20手势识别实验/README.md) |
| 9 | Hand Keypoint Detection | `hand_keypoint_detection.py` | 手掌关键点检测 | [实验10 手掌关键点检测](../标准例程-CanMV版/AI类实验例程/实验10%20手掌关键点检测实验/README.md) |
| 10 | Hand Keypoint Class | `hand_keypoint_class.py` | 手掌关键点分类 | [实验11 手掌关键点分类](../标准例程-CanMV版/AI类实验例程/实验11%20手掌关键点分类实验/README.md) |
| 11 | Dynamic Gesture | `dynamic_gesture.py` | 动态手势识别（时序） | [实验12 动态手势识别](../标准例程-CanMV版/AI类实验例程/实验12%20动态手势识别实验/README.md) |
| 12 | Space Resize | `space_resize.py` | 局部放大器 | [实验13 局部放大器](../标准例程-CanMV版/AI类实验例程/实验13%20局部放大器实验/README.md) |
| 13 | Finger Guessing | `finger_guessing.py` | 猜拳游戏 | [实验14 猜拳游戏](../标准例程-CanMV版/AI类实验例程/实验14%20猜拳游戏实验/README.md) |
| 14 | Object Detect | `object_detect_yolov8n.py` | 物体检测（YOLOv8n） | [实验15 物体检测](../标准例程-CanMV版/AI类实验例程/实验15%20物体检测实验/README.md) |
| 15 | Segmentation | `segment_yolov8n.py` | 物体分割（YOLOv8n-seg） | [实验16 物体分割](../标准例程-CanMV版/AI类实验例程/实验16%20物体分割实验/README.md) |
| 16 | Licence Det | `licence_det.py` | 车牌检测 | [实验17 车牌检测](../标准例程-CanMV版/AI类实验例程/实验17%20车牌检测实验/README.md) |
| 17 | Licence Det Rec | `licence_det_rec.py` | 车牌号识别 | [实验18 车牌号识别](../标准例程-CanMV版/AI类实验例程/实验18%20车牌号识别实验/README.md) |
| 18 | Keyword Spotting | `keyword_spotting.py` | 语音关键词唤醒 | [实验19 关键词唤醒](../标准例程-CanMV版/AI类实验例程/实验19%20关键词唤醒实验/README.md) |
| 19 | Self Learning | `self_learning.py` | 自学习分类（现场采样训练） | [实验20 自分类学习](../标准例程-CanMV版/AI类实验例程/实验20%20自分类学习实验/README.md) |

> [!NOTE]
> 标准例程里的[实验4 人脸识别](../标准例程-CanMV版/AI类实验例程/实验4%20人脸识别实验/README.md)未收录进 AI Hub（AI Hub 里没有 `face_recognition` 演示）。想跑人脸识别，请直接运行标准例程那份。

---

## 8. 部署与运行

综合例程**不是**在 IDE 里点运行就能看的（它依赖 `/sdcard` 下的一整套资源目录），正确做法是把资源拷进开发板存储。官方说明（见根目录 `readme.txt`）："请复制到 CanMV 盘的 sdcard 文件夹下"。

### 第 1 步：选对版本文件夹

先看你的 CanMV 固件版本（IDE 连接后底部会显示），然后：

- 固件 **v1.2** → 用 `CanMV v1.2/`
- 固件 **v1.3** → 用 `CanMV v1.3/`

（两者区别只在 RTC 的一行 API，见[第 11 节](#11-v12-与-v13-的区别)，选错会导致设置里的时间写入失败。）

### 第 2 步：拷贝资源到开发板

把选定版本文件夹里的内容拷到开发板的 `/sdcard/`（即 CanMV 盘符对应的 sdcard 目录）：

1. 将 **`CanMV Sample` 整个文件夹** 拷贝到 `/sdcard/`，得到 `/sdcard/CanMV Sample/`（含 `APP`、`Photos`、`Fonts`）；
2. 将该版本根目录下的 **`main.py`** 拷贝为 **`/sdcard/main.py`**（作为开机入口脚本）。

拷贝完成后开发板的目录应类似：

```
/sdcard/
├── main.py                 ← 桌面 GUI 开机脚本
├── CanMV Sample/           ← 资源工程（名字不能改）
│   ├── APP/  Photos/  Fonts/  main.py
├── examples/               ← AI 模型与库（由固件/标准例程提供，AI Hub 需要）
│   ├── kmodel/  utils/
└── libs/                   ← PipeLine / AIBase / AI2D 等（固件提供）
```

> [!WARNING]
> 若 `/sdcard/examples/`、`/sdcard/libs/` 不存在，AI Hub 的演示无法运行。请确认你的 TF 卡 / 板载存储是**烧录了完整 CanMV 固件 + 例程资源**的，而不是只拷了本工程。

### 第 3 步：重启运行

断开 IDE、给开发板重新上电（或按 RESET）。开发板会执行 `/sdcard/main.py`，屏幕显示**锁屏界面**——上滑解锁即可进入桌面。

### 第 4 步：运行 AI 演示 / 返回桌面

- 进 **AI Hub** → 选演示 → "Reboot to Run" → 长按 KEY0 可取消，点 Confirm 重启进入演示；
- 演示运行时按演示自身约定的按键退出 / 复位；
- **想回到桌面系统**：把 `/sdcard/CanMV Sample/main.py` 重新拷贝覆盖 `/sdcard/main.py`，再重启即可。

---

## 9. 实验效果

**成功现象：**

1. 上电后屏幕显示锁屏页：壁纸 + 超大时钟（默认 08:00）+ 底部 "Swipe up to open" 上下浮动动画；
2. 手指从底部向上滑动超过约 1/3 屏高，锁屏页上移消失，进入桌面；
3. 桌面顶部状态栏显示时间，中间是应用图标（可左右翻页），底部 Dock 有 4 个常驻图标；
4. 点击 Calculator / Clock / Photos / Settings / Freeform / Tester 等图标，对应 APP 以放大动画打开，可触摸操作；
5. Tester 里拖动滑块，板上红 / 蓝 LED 亮度实时变化，蜂鸣器音量和音调随之改变；
6. Settings → Date & Time 修改时间后，锁屏和状态栏时钟同步更新；
7. 打开 AI Hub，左侧是演示列表，点某项右侧显示预览图和 "Reboot to Run" 按钮，确认后重启进入摄像头 AI 画面；
8. 任意界面按下 **KEY0** 返回锁屏。

**失败现象排查：**

| 现象 | 可能原因 |
| --- | --- |
| 黑屏 / 无界面 | `CanMV Sample` 未拷到 `/sdcard/`，或文件夹被改名，或没拷 `main.py` 到 `/sdcard/main.py` |
| 界面文字变成方块 / 缺字 | `Fonts/` 字库缺失或路径不对 |
| 图标 / 壁纸不显示 | `APP/icons/`、`Photos/` 缺失 |
| AI 演示报 kmodel 错误 | `/sdcard/examples/kmodel/`、`/sdcard/libs/` 缺失（见第 8 节） |
| 设置时间无效 / 报错 | **固件版本与工程版本不匹配**（v1.2/v1.3 的 RTC API 不同，见第 11 节） |
| 反复重启 | 供电不足，换 5V/2A 以上电源 |

---

## 10. 注意事项

> [!WARNING]
> 1. **文件夹名 `CanMV Sample` 不可改**：资源路径在代码里写死为 `/sdcard/CanMV Sample/`。
> 2. **版本要选对**：v1.2 / v1.3 工程的 RTC 初始化 API 不同，选错会导致设置时间失败（详见第 11 节）。
> 3. **运行 AI 演示会覆盖 `/sdcard/main.py`**：即覆盖桌面本身。回桌面需重新拷贝 `CanMV Sample/main.py`。
> 4. **AI Hub 依赖外部模型和库**：`examples/kmodel`、`examples/utils`、`libs` 必须由固件 / 标准例程资源提供，本仓库不含。
> 5. **必须脱机运行**：本工程靠 `/sdcard/main.py` 开机自启，不适合在 IDE 里直接"运行当前脚本"（资源路径、重启逻辑都对不上）。
> 6. **供电与散热**：GUI + 摄像头 + KPU 负载大，用 5V/2A 以上电源，别遮散热面。
> 7. **桌面第 2 页的部分图标是占位装饰**（Books/Files/Calendar/Health/Music/Weather 等），点击无反应属正常。
> 8. 修改代码前**先备份** `main.py`；工程较大，一处语法错误就会导致整个 GUI 起不来。

---

## 11. v1.2 与 v1.3 的区别

两个版本的 **AI Hub 演示脚本、资源文件完全相同**，**唯一区别**在 `CanMV Sample/main.py` 的 `ClockManager.set_time()`——RTC 初始化参数的排列不同：

```python
# CanMV v1.2 固件：元组里第 4 个位置是 weekday(0)
self.rtc.init((year, month, day, 0, hour, minute, second, microsecond))

# CanMV v1.3 固件：去掉了 weekday，直接 年月日时分秒
self.rtc.init((year, month, day, hour, minute, second, microsecond))
```

**为什么会这样？** CanMV 固件在 v1.2→v1.3 之间改了 `RTC.init()` 的参数格式（是否包含星期字段）。用错版本，写进 RTC 的时间会错位（比如把"星期"当成"小时"），导致设置里的时间保存后不对。

> [!TIP]
> 不确定自己固件版本时：连接 IDE 看底部固件版本号；或者两个都试——能在"设置 → Date & Time"里正确保存并显示时间的，就是对的版本。

---

## 12. 常见问题 FAQ

**Q1：拷进去重启后还是黑屏 / 没进入桌面？**
A：按顺序检查：① `/sdcard/main.py` 是否存在且是桌面脚本；② `/sdcard/CanMV Sample/` 是否完整（APP、Photos、Fonts 三个子目录都在）；③ 文件夹名有没有被改（必须叫 `CanMV Sample`）；④ 供电是否足够。

**Q2：进了 AI Hub 点演示，重启后报错打不开模型？**
A：说明 `/sdcard/examples/kmodel/` 或 `/sdcard/libs/` 缺失。这些不由本仓库提供，需要用烧录了完整固件 + 例程资源的存储卡。参见[第 2.2 节](#22-ai-hub-演示额外用到的资源)。

**Q3：运行完 AI 演示，怎么回到桌面？**
A：把 `/sdcard/CanMV Sample/main.py` 重新拷贝覆盖 `/sdcard/main.py`，再重启。因为运行演示时它被覆盖成了演示脚本。

**Q4：设置里改了时间，锁屏时钟不变 / 报错？**
A：多半是固件与工程版本不匹配。确认你用的是对应 v1.2 或 v1.3 的工程（见第 11 节）。

**Q5：桌面上有些图标点了没反应？**
A：正常。只有 `app_dict` 里注册的 8 个 APP（Settings/Clock/Calculator/AI Hub/Photos/Freeform/Tester/Template）能打开，其余图标是占位装饰，留给你自己扩展。

**Q6：能像标准例程那样在 IDE 里直接运行调试吗？**
A：不建议。本工程依赖 `/sdcard` 下的资源目录和"重启切换脚本"机制，应脱机（拷贝到 `/sdcard/main.py`）运行。要调试单个 AI 算法，请用[标准例程](../标准例程-CanMV版/README.md)里对应的独立实验。

**Q7：`lv.task_handler()` 和 `gc.collect()` 为什么放在死循环里？**
A：`lv.task_handler()` 驱动 LVGL 的刷新、事件、动画，必须高频反复调用；`gc.collect()` 定期回收 MicroPython 内存，避免长时间运行内存碎片导致卡死。这是 LVGL + MicroPython 工程的标准写法。

---

## 13. 总结

- 本综合例程是一个基于 **LVGL** 的"类手机桌面系统"，把标准例程的零散知识点整合成**完整、可交互、产品级**的工程；
- 架构分三层：**硬件驱动层**（LCD/Touch/LED/Buzzer/Button/ClockManager 类）→ **GUI 层**（`GUI` 类，含锁屏、桌面、状态栏、Dock、AppManager 与 8 个 APP）→ **主循环**（`lv.task_handler()` + KEY0 返回锁屏）；
- **AI Hub** 通过"复制脚本到 `/sdcard/main.py` + `machine.reset()`"的方式，一键切换运行 19 个 AI 演示，每个演示都对应标准例程里的一个 AI 实验；
- 运行前务必：**选对固件版本（v1.2/v1.3）→ 把 `CanMV Sample` 拷到 `/sdcard/` 且不改名 → 把 `main.py` 拷成 `/sdcard/main.py` → 确保 `examples/`、`libs/` 存在 → 脱机重启**；
- 想扩展自己的 APP，从 `AppTemplate` 照着写，加进 `app_dict` 并在桌面添加图标即可。

学完这个综合例程，你就掌握了"**如何把嵌入式外设 + 图形界面 + AI 推理组织成一个完整产品**"的工程能力。下一步可以试着：改壁纸和图标做个性化桌面、新增一个自己的 APP、把某个 AI 演示和桌面深度结合（如人脸门禁面板）。

---

> 配套视频教程见正点原子官网（www.alientek.com）与技术论坛（www.openedv.com）。本综合例程源码与文档版权归原作者所有，仅供学习交流。
