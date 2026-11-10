# 实验20 自分类学习实验

> 本例程在正点原子 DNK230D（K230D BOX）开发板上实现"免训练"的物体分类：先用按键现场采集若干物体的特征存入 TF 卡"特征库"，之后开发板就能实时识别画面中央的物体属于哪一类并给出相似度分数——就像教小孩认东西一样，现学现用。难度：高级。

## 1. 例程简介

传统分类模型要"采集数据→电脑训练→转换 kmodel→拷回板子"，而本实验演示 **自学习（Self-Learning / Few-shot 分类）**：利用一个通用的特征提取模型 `recognition.kmodel`，把画面中央 400×400 区域里的物体压缩成一个特征向量；用户按下 KEY1 后，程序按预定义类别（默认"苹果、香蕉、梨"）逐类采集特征并以 .bin 文件存库；采集完成后，程序每帧计算当前物体特征与库中所有特征的**余弦相似度**，把相似度超过阈值的前 top_k 个结果（按类别去重）显示在屏幕上。

- **知识点**：特征向量（Embedding）、余弦相似度、阈值判决、top-k 排序、特征库文件读写（tobytes/frombuffer）、按键消抖、Ai2d crop+resize 预处理、注册/识别两阶段状态机。
- **难度**：高级（完整覆盖"建库→识别"闭环，涉及文件 IO、按键交互和多阶段逻辑）。
- **用途**：无需重新训练即可扩展新类别的场景，如分拣线新增品类、家庭物品识别玩具、教学演示"机器学习特征匹配"原理。

> [!NOTE]
> - 特征提取模型：`/sdcard/examples/kmodel/recognition.kmodel`，输入 224×224；
> - 特征库目录：`/sdcard/examples/utils/features/`；
> - **程序退出时（finally）会自动删除整个特征库目录**，即特征库是"一次性"的，每次运行都要重新注册。

## 2. 硬件连接与原理

### 2.1 硬件清单

| 硬件 | 说明 |
| --- | --- |
| CNK230DF 核心板 | 嘉楠 K230D 芯片（RISC-V 双核 + 6TOPS 算力 KPU） |
| DNK230D 底板 | 提供按键、LCD、摄像头、TF 卡槽 |
| 2.4 寸 MIPI LCD | 640×480，显示采集框、提示语和识别结果 |
| MIPI CSI 摄像头 | `Sensor(width=1280, height=960)` 采集图像 |
| 按键 KEY1 | 代码中映射为 **GPIO35**（`fpioa.set_function(35, FPIOA.GPIO35)`，上拉输入），触发特征采集 |
| TF 卡 | 存放固件、`recognition.kmodel` 和特征库 `features/` 目录 |

> [!WARNING]
> - 摄像头/LCD 排线插反会烧坏接口，务必断电插接、禁止带电插拔。
> - 特征库频繁写 TF 卡，请使用质量可靠的 TF 卡；运行中途直接断电可能导致 .bin 文件不完整。
> - KPU 持续推理发热属正常现象；供电不足会引起重启或按键误触发。

### 2.2 工作原理（通俗版）

1. PipeLine 输出 1280×960 的 AI 图，Ai2d 先 **crop** 出正中央 400×400 的区域（屏幕上对应一个黄色方框，提示"把物品放进框里"），再 **resize** 成 224×224 送入 KPU。
2. `recognition.kmodel` 是一个特征提取网络（去掉分类头的图像识别模型），输出一个高维**特征向量**——可以理解为该物体的"数字指纹"。`postprocess` 直接返回 `results[0][0]`。
3. **注册阶段**：按下 KEY1（10ms 消抖）后 `key_node=1`，程序把每一帧的特征向量写入 `features/类别名_序号.bin` 文件；每 `time_one=60` 帧换一个 .bin 文件，每个类别默认写 `features=2` 个文件；当前类别写满后自动切到下一类别并复位按键标志（需再次按键继续）。屏幕实时显示正在采集的文件名，如 `苹果_1.bin`。
4. **识别阶段**：所有类别注册完毕（`category_index >= len(labels)`）后，每帧遍历特征库所有 .bin 文件，用 `np.frombuffer` 还原特征向量，计算与当前特征的**余弦相似度**；相似度 > threshold(0.5) 的结果按类别去重（同类保留最高分），只保留得分最高的 top_k(3) 条，按分数从高到低在屏幕上显示 `类别 : 分数`。
5. 程序退出时 `finally` 删除所有 .bin 文件并移除 `features/` 目录。

## 3. 相关前置知识

- **特征向量（Embedding）**：神经网络把一张图压缩成的一串浮点数。相似物体的特征向量在空间中"距离近"，不同物体"距离远"。有了好的特征向量，分类可以不用重新训练——直接比距离即可。
- **余弦相似度**：两个向量夹角的余弦值，`dot(A,B) / (|A|·|B|)`，取值范围约 [-1,1]，越接近 1 越相似。代码中 `getSimilarity` 就是逐行实现这个公式（`sum(a*b)` 为点积，`np.sqrt(sum(a*a))` 为模长）。
- **阈值 threshold**：相似度低于 0.5 的匹配直接丢弃，防止"库里没有的物体"被强行归类。
- **top_k**：最多显示得分最高的 3 条匹配结果；代码用"插入后按分数降序排序、超出则弹出末位"维护这个小顶堆式列表。
- **按类别去重**：同一类别在库里有多个 .bin 特征，识别时同一类别只保留最高分那一条，避免屏幕被同一类别刷屏。
- **特征的持久化**：`feature.tobytes()` 把 numpy 数组按二进制写入文件；`np.frombuffer(data, dtype=np.float)` 读回还原。文件名 `类别_序号.bin` 中，`split("_")[0]` 即类别名——**类别名里不能含下划线**。
- **按键消抖**：机械按键按下瞬间会抖动，代码采用"检测到低电平 → `sleep_ms(10)` → 再次确认低电平 → 等待松开"的标准消抖流程。
- **Ai2d 执行顺序**：crop → resize（Ai2d 支持 crop/shift/pad/resize/affine，本例只用前两步之一组合）。
- **帧计数注册**：注册不是"按一次存一个"，而是按键后**每帧都存**，靠帧计数 `time_now // time_one` 决定写入哪个 .bin 文件；60 帧大约对应 2~4 秒（取决于帧率），期间可以缓慢转动物体让特征更全面。

## 4. 软件代码逻辑说明

### 4.1 程序整体流程

```
配置 KEY1（GPIO35，上拉输入）→ 创建 PipeLine（摄像头 + LCD）
→ 创建 SelfLearningApp（加载 recognition.kmodel、建 features/ 目录、计算 crop 位置）
→ 配置 Ai2d（crop 中央 400×400 + resize 到 224×224）
→ 主循环：检测按键 → 取帧 → 提取特征
   ├─ 注册阶段（category_index < 类别数）：按键后逐帧写 .bin，写满自动换类
   └─ 识别阶段：遍历特征库 → 余弦相似度 → 阈值过滤 → 类别去重 → top_k 排序 → 显示
→ 退出：删除 features/ 目录全部文件 → rmdir → deinit() → pl.destroy()
```

### 4.2 按键与参数配置

```python
fpioa = FPIOA()
fpioa.set_function(35, FPIOA.GPIO35)
key1 = Pin(35, Pin.IN, pull=Pin.PULL_UP, drive=7)  # 构建key1对象
key_node = 0                                        # 按键标志位（全局）
```

```python
kmodel_path="/sdcard/examples/kmodel/recognition.kmodel"
database_path="/sdcard/examples/utils/features/"
rgb888p_size=[1280,960]
model_input_size=[224,224]
labels=["苹果","香蕉","梨"]   # 要注册的类别，可自行增改
top_k=3                       # 最多显示3条识别结果
threshold=0.5                 # 相似度阈值
```

### 4.3 SelfLearningApp 初始化要点

```python
self.features = [2 for i in range(len(labels))]  # 每个类别注册2个特征文件
self.time_one = 60                                # 单个特征文件采集60帧
self.category_index = 0                           # 当前正在注册的类别索引
self.crop_w = 400; self.crop_h = 400              # 特征采集框宽高
self.crop_x = self.rgb888p_size[0]/2.0 - self.crop_w/2.0   # 居中
self.crop_y = self.rgb888p_size[1]/2.0 - self.crop_h/2.0
```

`data_init()` 负责创建 `features/` 目录（已存在则忽略异常）、把 crop 框换算成屏幕坐标（`crop_x_osd` 等）。

### 4.4 预处理与后处理

```python
def config_preprocess(self,input_image_size=None):
    ...
    self.ai2d.crop(int(self.crop_x),int(self.crop_y),int(self.crop_w),int(self.crop_h))
    self.ai2d.resize(nn.interp_method.tf_bilinear, nn.interp_mode.half_pixel)
    self.ai2d.build([1,3,ai2d_input_size[1],ai2d_input_size[0]],
                    [1,3,self.model_input_size[1],self.model_input_size[0]])

def postprocess(self,results):
    return results[0][0]   # 直接返回特征向量
```

### 4.5 主循环与按键消抖

```python
while True:
    os.exitpoint()
    with ScopedTiming("total",1):
        if key1.value()==0:            # 按键被按下
            time.sleep_ms(10)          # 消除抖动
            if key1.value()==0:        # 确认按下
                print('key1被按下')
                key_node = 1           # 开始特征采集
                while not key1.value():# 等待松开
                    pass
        img=pl.get_frame()
        res=sl.run(img)                # 提取特征向量
        sl.draw_result(pl,res)         # 注册或识别 + 绘制
        pl.show_image()
        gc.collect()
```

### 4.6 draw_result：注册阶段

```python
pl.osd_img.draw_rectangle(self.crop_x_osd,self.crop_y_osd,
                          self.crop_w_osd,self.crop_h_osd,
                          color=(255,255,0,255), thickness=4)   # 黄色采集框
if (self.category_index < len(self.labels)):
    if key_node == 0:
        # 提示：请将物品[苹果]放入框,按键开始特征采集：
        pl.osd_img.draw_string_advanced(...)
    if key_node == 1:
        self.time_now += 1
        # 每帧把特征写入 类别_序号.bin
        with open(self.database_path + self.labels[self.category_index] + "_"
                  + str(int(self.time_now-1)//self.time_one) + ".bin", 'wb') as f:
            f.write(feature.tobytes())
        if (self.time_now // self.time_one == self.features[self.category_index]):
            self.category_index += 1   # 该类别采集完成，切换下一类
            self.time_all -= self.time_now
            self.time_now = 0
            key_node = 0               # 复位，等待下一次按键
```

### 4.7 draw_result：识别阶段

```python
else:   # 所有类别注册完成
    results_learn = []
    list_features = os.listdir(self.database_path)
    for feature_name in list_features:
        with open(self.database_path + feature_name, 'rb') as f:
            data = f.read()
        save_vec = np.frombuffer(data, dtype=np.float)
        score = self.getSimilarity(feature, save_vec)   # 余弦相似度
        if (score > self.threshold):
            res = feature_name.split("_")               # 取类别名
            # 同类别只保留最高分；总数不超过top_k，按分数降序维护
            ...
    draw_y = 200
    for r in results_learn:
        pl.osd_img.draw_string_advanced(50, draw_y, 50,
            r["category"] + " : " + str(r["score"]), color=(255,255,0,0))
        draw_y += 50
```

### 4.8 相似度计算

```python
def getSimilarity(self,output_vec,save_vec):
    tmp = sum(output_vec * save_vec)          # 点积
    mold_out = np.sqrt(sum(output_vec * output_vec))   # 模长
    mold_save = np.sqrt(sum(save_vec * save_vec))
    return tmp / (mold_out * mold_save)       # 余弦相似度
```

### 4.9 退出清理（finally）

```python
stat_info = os.stat(database_path)
if (stat_info[0] & 0x4000):        # 判断是目录
    list_files = os.listdir(database_path)
    for l in list_files:
        os.remove(database_path + l)   # 删除全部特征文件
os.rmdir(database_path)                # 删除features目录
sl.deinit()
pl.destroy()
```

### 4.10 可调参数一览

| 参数 | 位置 | 作用 |
| --- | --- | --- |
| `labels=["苹果","香蕉","梨"]` | `__main__` | 自定义类别（名称中不要含 `_`），改几个类就注册几个类 |
| `threshold=0.5` | `__main__` | 调大更严格（宁缺毋滥），调小更宽松（易误判） |
| `top_k=3` | `__main__` | 屏幕最多显示的匹配条数 |
| `self.features=[2,...]` | `__init__` | 每类特征文件数，加大可提升鲁棒性但注册更久 |
| `self.time_one=60` | `__init__` | 每个特征文件的采集帧数 |
| `self.crop_w/crop_h=400` | `__init__` | 中央采集框大小 |

## 5. 运行效果

- **注册阶段屏幕**：黄色方框位于画面中央；框上方显示 `请将物品[苹果]放入框,按键开始特征采集：`。按下 KEY1 后提示变为 `请将待添加类别放入框内进行特征采集：苹果_0.bin`，序号随帧数递增；一个类别采完自动切换到下一类别提示，全部采完后进入识别阶段。
- **识别阶段屏幕**：黄色方框保留，屏幕左侧从 y=200 起逐行显示如 `苹果 : 0.8731`、`香蕉 : 0.4123`（仅分数 > 0.5 且前 top_k 名）。把不同水果依次放入框中，最高分类别应随之切换。
- **串口输出**：按键时打印 `key1被按下`；`ScopedTiming("total",1)` 打印每帧耗时；`print(res)` 被注释，打开可看到原始特征向量。
- **成功标志**：注册流程顺畅走完，识别阶段最高分结果与框内实物一致，换物体后结果随之改变。
- **失败现象排查**：
  - 按 KEY1 无反应 → 确认按的是与 GPIO35 对应的按键（代码注释中的 key1）；串口应打印 `key1被按下`。
  - 识别阶段什么都不显示 → 所有相似度都低于 0.5：注册时的物体与识别时差异太大（光照/角度/距离），重新注册或调低 `threshold`。
  - 结果分数都接近 → 特征区分度不足，确保注册时框内只有目标物体、背景尽量单一。
  - 第二次运行报目录错误 → 上次异常退出残留了 `features/` 目录，`data_init` 会自动复用；必要时手动删除 TF 卡上该目录。

## 6. 代码使用说明

1. **固件准备**：烧录与 CanMV IDE 版本匹配的 **CanMV K230D 固件**（正点原子出厂固件即可）。
2. **拷贝模型文件**：将正点原子资料中的模型拷贝到 TF 卡：
   - `/sdcard/examples/kmodel/recognition.kmodel`（特征提取模型）
   - 特征库目录 `/sdcard/examples/utils/features/` 由程序自动创建，无需手动拷贝。
3. **依赖库确认**：`libs.PipeLine`、`libs.AIBase`、`libs.AI2D` 位于 `/sdcard/app/libs/`；另用到固件内置的 `machine.Pin/FPIOA`、`ulab.numpy`、`nncase_runtime`、`image`、`aicube`（import 了但本流程未用到其接口）。无需第三方库。
4. **运行步骤**：
   - 打开 CanMV IDE → 连接开发板 → 打开本目录 `main.py` → 运行；
   - 把第一个物品（如苹果）放入屏幕黄框内 → 按 KEY1 → 等待屏幕提示的文件序号采完（期间可缓慢转动物体）；
   - 提示切换下一类别后，换物品再按 KEY1，重复直到所有类别注册完成；
   - 进入识别阶段后，把任意已注册物品放入框中，观察屏幕显示的类别与分数；
   - 停止运行：**程序会自动清空特征库**，下次运行需重新注册。

> [!WARNING]
> 想保留特征库供下次直接使用，需要注释掉 `finally` 中删除 `features/` 目录的代码段，并把识别阶段的进入条件（`category_index` 初值）调整为直接跳过注册——这是拓展练习之一。

## 7. 拓展思考

1. **持久化特征库**：修改 `finally` 清理逻辑与启动逻辑，让程序启动时若发现 `features/` 中已有 .bin 文件则直接进入识别阶段，实现"一次注册、多次使用"。
2. **扩展自己的类别**：把 `labels` 改成 ["笔","橡皮","尺子"] 等身边物品重新注册，测试类间差异大小对识别准确率的影响，并尝试调整 `threshold` 找到最佳值。
3. **多特征投票**：把 `self.features` 从 2 加大到 5，注册时从更多角度采集；识别时观察同类别多条特征的分数分布，思考"平均特征向量"或"多数投票"能否进一步提升稳定性。

## 8. 常见问题

**Q1：为什么退出程序后特征库没了？**
A：这是例程有意设计：`finally` 中会删除 `features/` 目录下所有 .bin 文件并 `os.rmdir`，保证每次实验都从干净状态开始。想保留请见"拓展思考 1"。

**Q2：按 KEY1 没反应，屏幕提示一直是"按键开始特征采集"？**
A：① 确认按的是 GPIO35 对应的按键（不同批次底板丝印可能不同，可用万用表或打印 `key1.value()` 验证）；② 按键需按下并保持到消抖确认（10ms）；③ 检查串口是否打印了 `key1被按下`——打印了但界面没变化属于显示刷新问题，重试运行。

**Q3：识别结果里分数超过 1 或者是负数？**
A：余弦相似度理论范围 [-1,1]。若特征向量未归一化且数值异常（如注册时框内全黑），可能出现怪异分数。确保注册和识别时框内都有清晰、被照亮的物体。

**Q4：注册"苹果"后识别"苹果"分数却很低？**
A：常见原因：① 注册时物体没有充满黄框，识别时距离/大小变了；② 光照变化大；③ 注册期间物体一直在动导致特征模糊。建议注册时物体尽量填满框、缓慢小幅转动，识别时保持相似的摆放距离。

**Q5：类别名可以随便取吗？**
A：基本可以（支持中文），但**不能包含下划线 `_`**——文件名 `类别_序号.bin` 靠 `split("_")` 解析类别名，含下划线会把类别名截断。也不能包含文件系统非法字符。

**Q6：能否同时识别框内多个物体？**
A：不能。模型只对中央 400×400 裁剪区域提取**一个**特征向量，一次只对应一个物体。多物体识别需要先做目标检测（如实验15）再对每个框分别特征化，可作为进阶改造方向。
