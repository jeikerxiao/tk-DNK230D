# 综合例程 · CanMV v1.3 工程

> 本文件夹是 **适配 CanMV 固件 v1.3** 的综合例程工程（LVGL 桌面系统 + AI Hub）。完整的系统介绍、架构讲解、APP 说明、AI Hub 演示索引、注意事项，请看上一层的 [综合例程总 README](../README.md)。**本页只讲"这个版本怎么部署运行"以及"它和 v1.2 的区别"。**

---

## 1. 我该用这个文件夹吗？

先确认你的 **CanMV 固件版本**（用 CanMV IDE 连接开发板后，底部状态栏会显示固件版本）：

- 固件是 **v1.3** → 用**本文件夹**（`CanMV v1.3/`）✅
- 固件是 **v1.2** → 请改用 [`CanMV v1.2/`](../CanMV%20v1.2/README.md)

> [!WARNING]
> 版本选错会导致"设置 → Date & Time"里保存的时间错乱（原因见[第 4 节](#4-v13-与-v12-的唯一区别rtc)）。其余功能（桌面、计算器、相册、AI Hub 等）两个版本表现一致。

---

## 2. 本文件夹内容

```
CanMV v1.3/
├── README.md          ← 本页
├── main.py            ← 开机入口脚本（内容 = CanMV Sample/main.py）
└── CanMV Sample/      ← 需整体拷贝到 /sdcard/ 的资源工程
    ├── main.py        ← GUI 主程序（整个桌面系统，3500+ 行）
    ├── APP/           ← 图标(icons)、设置(Settings)、测试(Tester)、AI Hub 脚本
    ├── Photos/        ← 相册 APP 的示例图片
    └── Fonts/         ← LVGL 字库(.bin)
```

---

## 3. 部署运行（4 步）

> 详细原理与避坑见 [综合例程总 README · 第 8 节](../README.md#8-部署与运行)。这里是速查版。

1. **拷资源**：把本文件夹里的 **`CanMV Sample` 整个文件夹**拷贝到开发板的 `/sdcard/`，得到 `/sdcard/CanMV Sample/`（文件夹名**不能改**，代码里写死了资源路径）。
2. **拷入口**：把本文件夹根目录下的 **`main.py`** 拷贝为 **`/sdcard/main.py`**（开机自启脚本）。
3. **确认 AI 依赖**（只有跑 AI Hub 演示才需要）：`/sdcard/examples/kmodel/`、`/sdcard/examples/utils/`、`/sdcard/libs/` 必须存在——它们由完整 CanMV 固件 / 标准例程资源提供，**本仓库不含**。
4. **重启**：断开 IDE、重新上电（或按 RESET）。屏幕出现**锁屏页**即成功，**上滑解锁**进入桌面。

**运行 AI Hub 演示 / 回桌面：**

- AI Hub → 选演示 → "Reboot to Run" → Confirm，开发板会重启进入该演示；
- ⚠️ 运行演示会把 `/sdcard/main.py` **覆盖成演示脚本**。想回桌面，把 `/sdcard/CanMV Sample/main.py` 重新拷回 `/sdcard/main.py` 再重启即可。

---

## 4. v1.3 与 v1.2 的唯一区别：RTC

两个版本的 AI Hub 脚本、资源文件**完全相同**，唯一差异在 `CanMV Sample/main.py` 的 `ClockManager.set_time()` 里 RTC 初始化参数：

```python
# ★ v1.3（本文件夹）：直接 年月日时分秒，不含 weekday
self.rtc.init((year, month, day, hour, minute, second, microsecond))
```

而 v1.2 的元组第 4 位多了一个 weekday 字段（填 0）：

```python
# v1.2：(year, month, day, 0, hour, minute, second, microsecond)
self.rtc.init((year, month, day, 0, hour, minute, second, microsecond))
```

CanMV 固件在 v1.2→v1.3 之间调整了 `RTC.init()` 的参数格式（去掉了星期字段）。**本文件夹已按 v1.3 的格式写好，v1.3 固件直接可用，无需改动。**

---

## 5. 常见问题

- **黑屏 / 不进桌面**：检查 `/sdcard/main.py` 是否存在、`/sdcard/CanMV Sample/` 是否完整且未改名、供电是否 ≥5V/2A。
- **设置时间无效**：确认固件确实是 v1.3；若是 v1.2 请改用 [`CanMV v1.2/`](../CanMV%20v1.2/README.md)。
- **AI 演示报模型错误**：`/sdcard/examples/kmodel/`、`/sdcard/libs/` 缺失，见第 3 步第 3 点。

更多问题见 [综合例程总 README · 第 12 节 FAQ](../README.md#12-常见问题-faq)。
