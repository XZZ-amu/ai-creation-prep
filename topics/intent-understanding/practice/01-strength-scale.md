# 实操 1：验证 strength 刻度感

> **要验证什么**：denoising strength 在 0.2 / 0.5 / 0.7 / 0.9 四档下，画家"动了多少"。亲眼看完，`feynman-cards.md` 框架 4（strength 刻度表）才从"我读过"变成"我相信"。
>
> **预期耗时**：环境搭建 30-60 分钟（一次性），实验本身 15 分钟。
>
> **预期结果**：4 张图横向对比，能看到从"几乎没变"→"画风偏移但人没变"→"开始变形"→"几乎不像原人"的连续变化。

---

## Part A：环境搭建（一次性，做完后续 3 次实操不再做）

### A.1 启动 ComfyUI

你说已经下完了 ComfyUI。先确认是哪个版本：

**情况 1：你下的是 ComfyUI Desktop（.dmg 文件，拖到 Applications）**
- 直接打开 Applications 里的 ComfyUI 图标
- 第一次启动会有**安装向导**：
  - GPU 选 **MPS**（M2 芯片必须选这个，不要选 CPU 也不要选 CUDA）
  - 安装目录选一个空间充足的位置（**至少留 10GB 空闲**），记住这个目录路径，后面要往这里放模型
  - 等待向导完成（会自动装 PyTorch nightly + 依赖，可能 5-10 分钟）

**情况 2：你下的是源码（git clone 或 zip 解压）**
- 这条路更折腾，建议你**关掉这个版本，去下 Desktop 版**：https://www.comfy.org/download
- Desktop 版是官方 2025 年推出的图形化包装，对设计师友好得多

**验证启动成功**：
- 浏览器自动打开（或手动访问 http://127.0.0.1:8188）
- 看到界面中央有一堆**节点方块连成的流程图**——这就是 ComfyUI 的 workflow 编辑器
- 如果看到默认 workflow（节点链：Load Checkpoint → CLIPTextEncode × 2 → EmptyLatentImage → KSampler → VAEDecode → SaveImage），就 OK

> **如果启动失败**：把错误信息贴回来，我帮你判断是 PyTorch 版本问题、内存问题，还是别的。M2 Mac 第一次启动最常见的坑是 PyTorch 不是 nightly 版。

---

### A.2 下载底模

你的目标是跑 SD 1.5 底模。推荐 **Realistic Vision V6.0 B1**（写实风格，2025 年仍在更新）。

**下载步骤**：

1. 打开浏览器访问：https://civitai.com/models/4201/realistic-vision-v60-b1
2. 第一次访问 Civitai 需要**注册账号**（免费，邮箱注册即可）
3. 找到右侧的下载按钮——选 **fp16 版本**（约 2GB），不要下全精度版（4GB，对 16GB 内存太挤）
4. 下载后文件名类似：`realisticVisionV60B1_v51VAE.safetensors`

**放到哪里**：

ComfyUI Desktop 的模型目录是你向导里选的"安装目录"下的 `models/checkpoints/`。

具体怎么找：
- 打开 Finder，按 **Cmd+Shift+G**（前往文件夹）
- 输入你向导里设置的安装目录路径，例如 `~/Documents/ComfyUI`
- 进入 `models/` → `checkpoints/`
- 把刚下载的 `.safetensors` 文件**拖进这个文件夹**

> **注意**：不要放进 `/resource/ComfyUI` 或别的奇怪位置——ComfyUI 更新时会清空那些位置。

---

### A.3 让 ComfyUI 识别新模型

回到 ComfyUI 浏览器界面：

1. 找到画布上的 **Load Checkpoint 节点**（最左边的那个绿色框，标着 "Load Checkpoint"）
2. 点节点上的下拉菜单（默认显示某个模型名）
3. 如果**只看到默认模型，没看到 Realistic Vision**：
   - 点界面右下角的**刷新按钮**（圆形箭头图标），或按 **R**
   - 再点下拉菜单，应该能看到 `realisticVisionV60B1_v51VAE.safetensors`
4. 选中它

**还看不到？**
- 检查文件路径：是不是真的放在 `models/checkpoints/` 下
- 检查文件名后缀：必须是 `.safetensors` 或 `.ckpt`
- 完全重启 ComfyUI

---

### A.4 第一张图：跑通文生图（建立信心）

不要急着做 strength 实验，**先确保你能出图**。

1. 找到 **CLIPTextEncode 节点**（有两个——上面的接 KSampler 的 positive 输入，下面的接 negative 输入）
2. 在上面的（positive）节点的文本框里写：`a cat sitting on a windowsill, soft lighting, photorealistic`
3. 下面的（negative）节点保留默认（通常是 `text, watermark` 之类）或留空
4. 找到 **EmptyLatentImage 节点**：
   - width: **512**
   - height: **512**
   - batch_size: **1**
5. 找到 **KSampler 节点**：
   - seed: **任意数字**（先不锁，第一张随便跑）
   - steps: **20**
   - cfg: **7**
   - sampler_name: **euler**（默认即可）
   - scheduler: **normal**（默认即可）
   - denoise: **1.0**（文生图默认就是 1.0，等下做图生图才会改）
6. 点界面右下角的 **Queue Prompt** 按钮（或按 **Ctrl+Enter**）

**预期**：30-60 秒后，**SaveImage 节点**里出现一张猫的图。

**输出文件位置**：安装目录下的 `output/` 文件夹（带时间戳的 PNG）。

> **如果 30 秒过去画面没动静**：看 ComfyUI 终端的日志（Desktop 版有专门的"Logs"窗口），通常是 MPS 内存不足。先关掉所有别的应用，再试一次。
>
> **如果跑了几分钟还没出图**：第一次跑模型加载比较慢（要把 2GB 模型读进内存），属于正常。第二次会快很多。

---

## Part B：strength 刻度实验

环境搭好后，进入主实验。

### B.1 实验设计

**变量**：只改 KSampler 节点的 `denoise` 值
**控制变量**（这些必须全锁死）：
- 同一张原图
- 同一个 prompt
- 同一个 seed（随机种子）
- 同一个底模
- 同一个 steps / cfg / sampler

**实验组**：4 组
- 组 A：denoise = **0.2**
- 组 B：denoise = **0.5**
- 组 C：denoise = **0.7**
- 组 D：denoise = **0.9**

---

### B.2 切换到图生图 workflow

默认 workflow 是文生图。你要改成图生图——**只改 2 个地方**：

**改动 1：删除 EmptyLatentImage 节点**

- 用鼠标点中 EmptyLatentImage 节点
- 按 **Delete** 或 **Backspace**

**改动 2：加入 LoadImage + VAEEncode 节点**

- 在画布**空白处右键** → 弹出菜单
- 选 **Add Node** → **image** → **LoadImage**（加载图片节点出现）
- 再次右键空白处 → **Add Node** → **latent** → **VAEEncode**（编码到草稿空间的节点）

**接线**：

ComfyUI 的接线是**点住一个节点的输出端口（小圆点），拖到另一个节点的输入端口**。

需要建的连接：
1. **LoadImage 的 IMAGE 输出** → **VAEEncode 的 pixels 输入**
2. **Load Checkpoint 的 VAE 输出** → **VAEEncode 的 vae 输入**
3. **VAEEncode 的 LATENT 输出** → **KSampler 的 latent_image 输入**

完成后的画面应该是：
```
LoadImage ─→ VAEEncode ─→ KSampler ─→ VAEDecode ─→ SaveImage
                ↑
Load Checkpoint ─VAE→
```

---

### B.3 上传你要做实验的原图

1. 选一张**自拍照片**（半身或脸部特写最好）
2. 点 LoadImage 节点上的 **choose file to upload** 按钮
3. 选你的自拍 → 上传

**重要**：原图建议**正方形或接近正方形**，分辨率 **512×512 到 768×768** 之间。原图尺寸太大会爆内存。

> 如果原图太大，先在 macOS 预览（Preview）里 Cmd+A 全选 → 工具 → 调整大小 → 改成 512×512 → 导出。

---

### B.4 锁定所有变量

**这一步不做，整个实验作废**。

1. **Prompt（CLIPTextEncode positive）**：写一个目标风格描述
   - 推荐写：`Studio Ghibli anime style, soft lighting, warm colors, cinematic`
   - 写完之后**不要再动**

2. **Seed（KSampler 节点）**：
   - 找到 KSampler 上的 `seed` 字段
   - 看 `control_after_generate` 字段——把它改成 **fixed**（固定不变）
   - 否则每次跑完 ComfyUI 会自动换 seed，结果差异里就混着随机性

3. 其他参数保持不动：steps=20, cfg=7, sampler=euler, scheduler=normal

---

### B.5 跑 4 组对比

**组 A：denoise = 0.2**

1. KSampler 节点的 `denoise` 改成 **0.2**
2. Queue Prompt
3. 等结果出来，**重命名输出文件**为 `strength_0.2.png`（或自己记下来哪个对应哪个）

**组 B：denoise = 0.5**

1. KSampler 节点的 `denoise` 改成 **0.5**
2. Queue Prompt（其他全部不动）
3. 重命名输出为 `strength_0.5.png`

**组 C：denoise = 0.7**

1. denoise 改成 **0.7**
2. Queue Prompt
3. 重命名 `strength_0.7.png`

**组 D：denoise = 0.9**

1. denoise 改成 **0.9**
2. Queue Prompt
3. 重命名 `strength_0.9.png`

---

### B.6 横向对比

把 4 张图放在 macOS 预览里**同时打开**（Cmd 多选 4 张 → 双击）→ 切换到**联系印（Contact Sheet）**视图，或在 Figma / Keynote 里横向贴一排。

**重点观察**（按这个顺序看）：

1. **整体相似度** — 这张图整体上还像原图吗？
2. **人脸像不像本人** — 五官位置、脸型、眼睛颜色有没有变
3. **画风偏移程度** — 从"几乎没动"到"完全宫崎骏"的渐变
4. **细节保留** — 头发丝、衣服花纹、背景物体有没有被重画

---

## Part C：实验后的回填动作（5 分钟，但是关键）

打开 `feynman-cards.md`，找到 **框架 4：strength 刻度感** 那张表格。

在每一行旁边，**用一句话写下你亲眼看到的现象**。例如：

| Strength | 表上的预测 | 你看到的实际 |
|---|---|---|
| 0.2 | 几乎不动原图 | （你的描述：比如 "只是颜色稍微偏暖了，几乎没变"） |
| 0.5 | 保留主体重画细节 | （你的描述） |
| 0.7 | 构图认得出，内容大变 | （你的描述） |
| 0.9 | 几乎纯文生图 | （你的描述） |

**如果你看到的现象和预测不一致**——把卡片表格的预测改成你看到的版本。**亲眼看到的永远高于"读过的判断"**。

---

## Part D：可选的延伸实验（如果你还有兴趣）

跑完 4 组对比，如果你还想多看一些：

**延伸 1：换 prompt 风格再跑一遍**
- prompt 换成 `oil painting, Van Gogh style`
- 跑同样 4 个 strength
- 看看不同风格下，strength 刻度感是否一致

**延伸 2：换原图类型**
- 一张是脸部特写
- 一张是全身照
- 看看相同 strength 在不同景别下"动多少"差异

**延伸 3：测试 strength 极端值**
- denoise = 1.0：等同于纯文生图，原图完全不影响
- denoise = 0.05：几乎完全是原图
- 看看是否符合你的直觉

---

## 常见问题

**Q：跑出来的图全黑/全噪点**
- 内存不足，关闭其他应用，把分辨率降到 384×384

**Q：跑了 5 分钟还没出图**
- 第一次加载模型慢是正常的（2GB 模型要读进内存）
- 第二次跑会快很多
- 如果第二次还是这么慢，看 ComfyUI 日志确认是不是没用上 MPS

**Q：seed 锁了但每次结果还是不一样**
- 检查 `control_after_generate` 是不是 `fixed`
- 检查所有其他参数（包括 prompt 文本、cfg、sampler）有没有不小心改动

**Q：4 张图差异很小，看不出 strength 刻度**
- 可能是 prompt 写得太弱（"a person" 这种太空泛，画家在低 strength 下没改动方向）
- 把 prompt 写得更"远离原图"一点（如从写实自拍 → "oil painting" 或 "anime"），差异会明显

---

## 完成标志

跑完这次实验，你应该能用自己的话回答：

1. denoise 0.2 时画家做了什么？0.9 时呢？
2. 在我自己的这台机器、这张原图、这个 prompt 下，**denoise 等于多少时人脸开始走样**？
3. 如果产品需求是"风格化但保住像本人"，我会选 denoise 多少？

回答完这 3 个问题，就可以进入第 2 次实操（蒙版高/低 strength 对比）。
