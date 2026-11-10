# 实验15 拍照实验

> LCD 实时取景，按 KEY0 存 JPG、按 KEY1 存 BMP，双通道摄像头 + 按键快门。难度：初级

## 1. 例程简介

本例程在[实验12 摄像头实验](../实验12%20摄像头实验/README.md)的取景基础上增加"快门"功能：摄像头通道 0 输出 VGA 画面绑定到 LCD 实时取景，通道 1 配置为 **SXGAM（1280×960）RGB565** 高清输出专供拍照；按下 **KEY0** 从通道 1 抓拍一张并保存为 `/data/PHOTO/photo.jpg`，按下 **KEY1** 保存为 `/data/PHOTO/photo.bmp`。你将学会**摄像头双通道分工、snapshot 抓拍、图像保存（JPG/BMP）、os.mkdir 建目录**。难度为**初级**。

## 2. 硬件连接与原理

| 外设 | 引脚/资源 | 代码中的对象 |
| --- | --- | --- |
| MIPI CSI 摄像头（1280×960 采集） | MIPI CSI 接口 | `sensor` |
| 2.4 寸 MIPI LCD（ST7701，640×480） | MIPI DSI | `Display` |
| KEY0 按键（拍 JPG） | GPIO34（上拉输入，按下读 0） | `key0` |
| KEY1 按键（拍 BMP） | GPIO35（上拉输入，按下读 0） | `key1` |
| 存储分区 | 照片保存到 `/data/PHOTO/` | `img.save(...)` |

摄像头和屏幕出厂已连接，无需接线；按键为板载资源。

### 硬件原理（通俗版）

**为什么要两个通道？** 取景和拍照的需求不同：取景要流畅省资源（VGA 640×480 就够 LCD 显示），拍照要清晰（1280×960 高分辨率）。CanMV 摄像头支持**多通道输出**——同一份采集数据在芯片内部分成两路：

- **通道 0**：VGA + YUV420SP，通过 `Display.bind_layer` 硬件直通 LCD，做实时取景；
- **通道 1**：SXGAM（1280×960）+ RGB565，平时不占用显示资源，按下快门时才用 `sensor.snapshot(chn=CAM_CHN_ID_1)` 抓一帧出来保存。

**JPG vs BMP**：BMP 是无压缩位图，1280×960 RGB565 约 2.4MB，画质无损但占空间；JPG 是有损压缩，同尺寸只有几百 KB，日常照片都用它。`img.save()` 根据**文件扩展名**自动选择编码格式。

`/data/` 是固件提供的应用数据分区（与 TF 卡的 `/sdcard/` 不同），代码启动时会自动创建 `/data/PHOTO` 目录。

### 新手风险提醒

> [!WARNING]
> - 摄像头排线插拔必须**断电**操作；镜头避免触摸。
> - `/data` 分区容量有限，照片会**反复覆盖同名文件**（见常见问题 Q2），不会写满分区，但若改名保存大量照片请注意剩余空间。
> - 拍照瞬间不要断电，避免文件写一半损坏。

## 3. 相关前置知识

- **通道（chn / CAM_CHN_ID_1）**：摄像头输出的分路编号，`CAM_CHN_ID_0`、`CAM_CHN_ID_1` 是固件定义的通道常量；不写 `chn` 参数默认通道 0。
- **Sensor.VGA / Sensor.SXGAM**：固件预定义的分辨率常量，VGA=640×480，SXGAM=1280×960。
- **Sensor.YUV420SP / Sensor.RGB565**：像素格式。YUV420SP 省带宽适合视频链路；RGB565 每像素 2 字节（红 5 位绿 6 位蓝 5 位），适合保存成图片。
- **snapshot（抓拍）**：`sensor.snapshot(chn=...)` 从指定通道取出**当前这一帧**，返回一个 `image.Image` 图像对象，可绘图、可保存。
- **bind 直显**：`Display.bind_layer` 把通道 0 数据流硬件直通显示图层（原理见[实验12 摄像头实验](../实验12%20摄像头实验/README.md)）。
- **os.mkdir**：创建目录；目录已存在会抛异常，所以例程用 `try/except` 包住"已存在就跳过"。
- **按键轮询 + 延时**：每拍一张后 `time.sleep_ms(50)`，避免一次按键连拍多张。

## 4. 软件代码逻辑说明

流程：**FPIOA 映射按键 → 建照片目录 → 配置摄像头双通道 → 绑定显示 → 启动采集 → 主循环按键拍照 → finally 释放资源**。

### 第 1 步：按键初始化

```python
fpioa = FPIOA()
fpioa.set_function(34, FPIOA.GPIO34)
fpioa.set_function(35, FPIOA.GPIO35)

key0 = Pin(34, Pin.IN, pull=Pin.PULL_UP, drive=7)
key1 = Pin(35, Pin.IN, pull=Pin.PULL_UP, drive=7)
```

与前面实验相同：KEY0/KEY1 上拉输入，按下读 0。

### 第 2 步：创建照片目录

```python
    try:
        os.mkdir("/data/PHOTO")
    except Exception:
        pass
```

尝试创建 `/data/PHOTO` 目录；如果目录已存在，`mkdir` 抛出的异常被 `except` 吞掉（`pass`），程序继续——这是"确保目录存在"的惯用写法。

### 第 3 步：摄像头双通道配置

```python
    sensor = Sensor(width=1280, height=960) # 构建摄像头对象
    sensor.reset() # 复位和初始化摄像头

    sensor.set_framesize(Sensor.VGA)      # 设置帧大小VGA(640x480)，默认通道0
    sensor.set_pixformat(Sensor.YUV420SP) # 设置输出图像格式，默认通道0

    # 将通道0图像绑定到视频输出
    bind_info = sensor.bind_info()
    Display.bind_layer(**bind_info, layer=Display.LAYER_VIDEO1)

    # 设置通道1输出格式，用于图像保存
    sensor.set_framesize(Sensor.SXGAM, chn=CAM_CHN_ID_1)  # 输出帧大小SXGAM(1280x960)
    sensor.set_pixformat(Sensor.RGB565, chn=CAM_CHN_ID_1) # 设置输出图像格式，选择通道1
```

- 通道 0（不写 chn 的默认通道）：VGA + YUV420SP，`bind_info()` 取出数据流信息后绑定到 LCD 视频图层 1，实现实时取景。
- 通道 1（`chn=CAM_CHN_ID_1`）：SXGAM 1280×960 + RGB565，专供拍照。

### 第 4 步：初始化显示并启动

```python
    Display.init(Display.ST7701, width = 640, height = 480, to_ide = False)
    MediaManager.init()  # 初始化media资源管理器
    sensor.run()  # 启动sensor
```

LCD 以 640×480 初始化，`to_ide=False` **不回传 IDE 预览**（画面只在 LCD 上看，省 USB 带宽）。`MediaManager.init()` 后 `sensor.run()` 启动连续采集。

### 第 5 步：主循环——按键快门

```python
    while True:
        os.exitpoint() # 检测IDE中断
        # 读取按键状态，并做相应的按键解释
        if key0.value() == 0:
            img = sensor.snapshot(chn=CAM_CHN_ID_1) # 从通道1捕获一张图
            img.save("/data/PHOTO/photo.jpg")
            print("snapshot success") # 提示照片保存成功
            time.sleep_ms(50)
        if key1.value() == 0:
            img = sensor.snapshot(chn=CAM_CHN_ID_1) # 从通道1捕获一张图
            img.save("/data/PHOTO/photo.bmp")
            print("snapshot success") # 提示照片保存成功
            time.sleep_ms(50)
        time.sleep_ms(10)
```

- `sensor.snapshot(chn=CAM_CHN_ID_1)`：从通道 1 抓取一帧 1280×960 的 RGB565 图像，返回 `image.Image` 对象。
- `img.save(路径)`：按扩展名保存，`.jpg` 自动 JPEG 压缩编码，`.bmp` 无压缩保存。
- 拍照后 `sleep_ms(50)` 防连拍，循环末尾 `sleep_ms(10)` 控制轮询节奏。

### 第 6 步：异常与资源释放

```python
except KeyboardInterrupt as e:
    print("user stop: ", e)
except BaseException as e:
    print(f"Exception {e}")
finally:
    if isinstance(sensor, Sensor):
        sensor.stop()
    Display.deinit()
    os.exitpoint(os.EXITPOINT_ENABLE_SLEEP)
    time.sleep_ms(100)
    MediaManager.deinit()
```

标准的媒体例程收尾：停摄像头 → 关显示 → 释放媒体内存（同实验 12）。

### 关键 API 小结

| API | 作用 |
| --- | --- |
| `sensor.set_framesize(Sensor.SXGAM, chn=CAM_CHN_ID_1)` | 配置通道 1 输出 1280×960 |
| `sensor.set_pixformat(Sensor.RGB565, chn=CAM_CHN_ID_1)` | 配置通道 1 像素格式 |
| `sensor.snapshot(chn=CAM_CHN_ID_1)` | 从通道 1 抓拍一帧，返回图像对象 |
| `img.save(path)` | 保存图像，按扩展名选择 JPG/BMP 编码 |
| `os.mkdir(path)` | 创建目录 |

### 可调参数

- 保存路径 `/data/PHOTO/photo.jpg`：改到 TF 卡 `/sdcard/PHOTO/` 也可以（目录需先创建）。
- 通道 1 分辨率 `Sensor.SXGAM`：可换成其他常量改变照片尺寸。
- 防连拍延时 `time.sleep_ms(50)`：加大则两次拍照间隔更久。

## 5. 运行效果

- **成功现象**：
  1. LCD 出现实时取景画面（**IDE 帧缓冲区无画面**，因为 `to_ide=False`）；
  2. 按 KEY0：终端打印 `snapshot success`，`/data/PHOTO/photo.jpg` 生成（约几百 KB）；
  3. 按 KEY1：终端打印 `snapshot success`，`/data/PHOTO/photo.bmp` 生成（约 2.4MB）；
  4. 照片内容就是按键瞬间镜头看到的 1280×960 画面，比 LCD 取景更清晰。
- **失败现象排查**：
  1. 报 snapshot/save 异常 → 确认 `/data/PHOTO` 目录创建成功（首次运行代码会自动建）。
  2. 取景黑屏 → 摄像头或 LCD 排线问题，断电重插；对照实验 12 排查。
  3. 按键无反应 → GPIO34/35 映射遗漏；确认按的是 KEY0/KEY1。
  4. 按一次拍了好几张 → 拍照后的 `time.sleep_ms(50)` 被删了。

## 6. 代码使用说明

1. 确认摄像头、LCD 已连接；用 **CanMV IDE** 打开本目录 `main.py`。
2. USB Type-C 连接开发板，点击**连接**、**运行**，对着 LCD 取景，按 KEY0/KEY1 拍照。
3. 固件要求：**CanMV K230D 固件**，版本与 IDE 匹配。
4. **不需要 TF 卡**（照片默认存 `/data/PHOTO/`，该分区在板载存储上）；**不需要模型文件**。
5. 取出照片：可在 IDE 交互终端用文件操作读出，或改写保存路径到 `/sdcard/`（需插 TF 卡）后用读卡器取。

> [!NOTE]
> 想同时在 IDE 里看取景画面，可把 `to_ide = False` 改成 `True`，代价是占用 USB 带宽、可能影响流畅度。

## 7. 拓展思考

1. **连号照片**：用全局计数变量把文件名改成 `photo_1.jpg`、`photo_2.jpg`……（`"photo_%d.jpg" % n`），不再覆盖旧照片，做一台真"相机"。
2. **拍照水印**：抓拍后先用 `img.draw_string_advanced()` 在图像上写入 RTC 时间（结合[实验7 RTC实验](../实验7%20RTC实验/README.md)）再保存，制作带时间戳的照片。
3. **定时连拍**：去掉按键判断，改为每 3 秒自动抓拍一张存盘，配合拓展 1 的连号文件名，做一个简易监控抓拍器；注意监控 `/data` 或 TF 卡的剩余空间。

## 8. 常见问题

**Q1：为什么拍照用通道 1，不直接从显示的通道 0 抓图？**
A：通道 0 只有 VGA（640×480），抓出来照片分辨率低；通道 1 配置成 SXGAM（1280×960），照片清晰得多。两通道各司其职——"预览用低清省资源，拍照用高清保画质"，这正是多通道设计的意义。

**Q2：每次拍照都存成同一个 `photo.jpg`，之前的照片呢？**
A：被**覆盖**了。`img.save()` 写同名文件会直接替换旧内容，所以板上永远只有最近一张 JPG 和一张 BMP。想保留历史请按拓展思考 1 给文件名加序号。

**Q3：JPG 和 BMP 该选哪个？**
A：日常记录选 JPG：体积小（压缩约 10 倍）、通用性好，损失一点画质；需要无损原图或后续做像素级处理时选 BMP。本实验两个按键各存一种，方便你对比同一场景两种格式的文件大小。

**Q4：`to_ide = False` 是不是写错了？我在 IDE 里看不到画面。**
A：不是错误，是有意为之。取景画面通过 bind 直显在 LCD 上，关闭 IDE 回传可以节省 USB 带宽、让系统更流畅。想在 IDE 看画面把它改成 `True` 即可。

**Q5：按 KEY0 后打印了 `snapshot success`，但怎么确认文件真的存进去了？**
A：在 IDE 交互终端执行 `import uos; uos.listdir("/data/PHOTO")`，能看到 `photo.jpg`；再用 `uos.stat("/data/PHOTO/photo.jpg")` 查看文件大小（第 7 个元素，JPG 应为几十~几百 KB）。文件操作详见[实验13 文件读写实验](../实验13%20文件读写实验/README.md)。
