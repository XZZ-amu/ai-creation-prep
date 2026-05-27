# 批量生成的风格统一：怎么让"每一张都是同一只猫、同一种画风"

> 写作说明：本文是 v2 范式的延伸——沿用 [intent-understanding/01-mechanism-v2.md](../intent-understanding/01-mechanism-v2.md) 的"画室五人组"比喻体系（翻译官 / 画家 / 草稿世界 / 骨架图 / 风格手册），并在画室里**新增两个角色**来讲清楚批量风格统一的事。如果还没读过 v2，建议先读那篇。和 [iteration-precision/01-mechanism.md](../iteration-precision/01-mechanism.md) 是姊妹篇——那篇讲"改一张图怎么不动其他"，这篇讲"批量生成怎么不串味"。

---

## 一句话总结

**让 AI 批量出图保持"同一个角色 / 同一种画风"，不是让画家更聪明，而是给画家一个"参考凭证"——一张图、一个 ID 向量、一本随身手册、一段共享记忆——让他在每一张新图里都能认出"哦还是这只猫"。**
工具箱里有六类东西：**给画家递一张参考图（IP-Adapter / Reference-only）**、**给画家发一本随身手册（LoRA / DreamBooth）**、**塞一张身份证给画家（PhotoMaker / InstantID / PuLID）**、**让画家边看原图边画新图（FLUX Kontext 类 in-context reference）**、**把多张图绑在一起画（StoryDiffusion 共享自注意力）**、**视频专属的 ID 注入（ConsisID）**。
每一类都在解决"画家怎么记住这个角色"这个问题——记忆方式不同，代价不同，能保住的东西也不同。

---

## 贯穿案例

整篇文档围绕这一个场景走完全程：

> 用户给一张参考图：**他家那只断了一根胡须、左耳缺一小角的橘猫**。
> 现在要批量生成 10 张图：**这只橘猫在咖啡馆窗边、在篮球场、在月球、在老北京胡同、在赛博朋克霓虹下**……每一张都是**同一只猫**——胡须断在同一边，耳朵缺角的方向一致，毛色和体型不漂。

每种机制讲完都会回答同一个问题：**这种机制下，10 张图里这只猫还能被认出来吗？哪里会失手？**

---

## 为什么 t2i 模型天生不会"批量保住同一个角色"

先把问题讲透。在 v2 里，prompt 在草稿世界（latent space）对应的是一片"区域"——里面装着所有可能的"戴红围巾的橘猫"。10 个不同的 seed = 走到这片区域里的 10 个不同坐标 = 10 只长得**都像橘猫但都不是同一只**橘猫。

**这是 t2i 模型的本质特性，不是 bug**：模型存的是"橘猫这个类别长什么样"，不是"用户家那只断胡须橘猫长什么样"。**用文字描述无论写多细，都没法把"这一只"和"那一类"分开**——这是为什么任何一种"批量风格统一"方案，本质上都在做同一件事：**给画家额外塞一个"非文字的指针"，把生成区域从"橘猫这一类"收窄到"这只橘猫"。**

> **比喻精确度补丁**（v2 扩展）：v2 里说"prompt 在地图上指出目标区域"，给人一种"prompt 写够细就能定位"的错觉。**事实是 prompt 再细，定位精度也是类别级，不是个体级**。要个体级定位必须给画家额外的视觉证据——这就是接下来六种机制都在做的事。

接下来要在画室里**新增两个角色**：

| 新增角色 | 对应概念 | 干什么 |
|---|---|---|
| **画家身边的图像档案员** | Image encoder（CLIP-Vision / ArcFace / VAE encoder） | 把参考图压成画家能用的"凭证向量" |
| **画家的助理画师** | Reference U-Net pass / shared self-attention | 同时画一张"参考画"，让主画家随时偷看 |

这两个角色在 v2 里没出场——v2 的画家只听翻译官给的文字。一旦输入里多了**参考图**或**多张图要互相一致**，这两个角色就上场了。

---

## 机制 1：IP-Adapter ——递一张图给画家当 prompt

### 干什么

用户给一张参考图（橘猫照片）+ 文字 prompt（"在咖啡馆"）。**图像档案员**用 CLIP 视觉编码器把这张图压成一组图像 token，画家在画的时候除了听翻译官给的文字 token，**再开一路注意力听这组图像 token**——这就是 **decoupled cross-attention**（解耦交叉注意力）：文字一条注意力通道，图像一条注意力通道，两条通道的输出加在一起。

权重很轻——基础版只有 22M 参数，却能比肩全模型 fine-tune 的效果。挂在 SD/SDXL/FLUX 上都能用。

### 比喻不准在哪

"图当 prompt"听起来像画家"看到了"参考图——其实**画家看到的是 22 个被压扁的图像 token**（IP-Adapter base 用 CLIP global feature；IP-Adapter Plus 用 patch features，token 更多、细节更密）。这意味着：

- **风格、氛围、整体颜色调性能传过去**——这些在 CLIP 全局特征里能编码；
- **像素级的特定细节（断在右边的胡须、耳朵缺的那一小角）大概率传不过去**——CLIP 训练时就是为了对齐"这是一只猫"这种语义，不是为了记住一只猫的私有特征；
- **强度可调**：`ip_adapter_scale=1.0` 时图像信号最强但容易压住文字 prompt（用户写的"在咖啡馆"被忽略，输出就是参考图本身）；`0.5` 是平衡点；`0.3` 以下文字主导。

### IP-Adapter Plus / FaceID 是怎么"加细节"的

- **IP-Adapter Plus**：把 CLIP 的 patch tokens 全部喂给画家（不只是全局向量），细节更密；用 ViT-H 而不是 ViT-L，分辨率更高。
- **IP-Adapter FaceID**：**不用 CLIP，改用 InsightFace 抽人脸 ID embedding**——人脸识别专门训过，对身份比 CLIP 敏感得多。但只对人脸有效，对猫狗、物体没用。
- **IP-Adapter FaceID Plus**：FaceID 的人脸 ID + CLIP 的 patch features 一起喂——身份更稳，但风格控制变弱（因为 CLIP 那一路又把"长得像参考图"的压力加回来了）。

### 回到我们的橘猫

- 用户上传断胡须橘猫照片 + prompt "in a cafe by the window"，scale=0.6；
- ✅ 画风、毛色调性、橘猫的整体感觉能保住——10 张图都是橘猫，颜色相近；
- ⚠️ 断胡须、缺耳角这种**个体级特征**保不住——CLIP 把它压成"橘猫"了；
- ⚠️ scale 太高（>0.8）：每张图都像参考图原图，"咖啡馆"的文字指令被压住，10 张图都是橘猫坐在原参考图的环境里；
- ⚠️ scale 太低（<0.4）：橘猫感都散了，画风也不像。

### 用户视角的失败现象

- **"看着是橘猫，但不是我家那只"**——CLIP 编码的是类别特征，不是个体特征。
- **"参考图风格压过了我的 prompt"**——scale 太高。
- **"换了背景人物的脸就崩了"**——人脸要专门用 FaceID 那一档，base IP-Adapter 对脸不够细。

---

## 机制 2：Reference-only ——让助理画师陪画

### 干什么

不要参考图编码、不要新模型、不要训练。**直接劫持画家的自注意力（self-attention）**：让"助理画师"也跑一遍参考图的去噪过程，把每一层 self-attention 的 K/V（key/value）缓存下来；主画家画新图时，每一层 self-attention 的 K/V 都**和助理画师的拼起来**——画家看自己的画时，余光能扫到助理那张参考画。

公式上：`Attention(Q_gen, [K_gen; K_ref], [V_gen; V_ref])`

`style_fidelity` 调整两张画的影响力比例，0-1 之间。

### 比喻不准在哪

"助理画师陪画"听起来像两个画家协作——其实**只是一个机械的注意力拼接**。助理画师没有"判断力"，只是把它每一层的中间状态摊开供主画家查看。这意味着：

- **风格、构图、纹理倾向能传过去**——这些都体现在 self-attention 的中间特征里；
- **依然不保个体细节**——比 IP-Adapter 更弱，因为没有专门训过的"图像通道"；
- **训练成本零**：纯推理时技巧。代价是稳定性低于 IP-Adapter。

### 回到橘猫

- 用户给参考图，prompt "在赛博朋克霓虹下"；
- ✅ 整体氛围、画风、橘色调能传过去；
- ⚠️ 个体特征（断胡须）肯定丢；
- ⚠️ 在 DiT 时代（FLUX、SD3）作用减弱——DiT 的 attention 模式和 UNet 不同，reference-only 这种 hack 更难直接套用，社区主流是直接用 FLUX Kontext（机制 4）。

### 用户视角的失败现象

- **"风格能传，但角色对不上"**——这是 reference-only 的本职定位，本来就不是给个体一致性用的。
- **"在新模型上没在 SD 1.5 上好用"**——UNet 时代的 trick，DiT 时代退潮。

**Reference-only 今天的位置**：作为零成本的"风格借鉴"工具还在用，但**不是批量角色一致性的主流方案**——主流是 IP-Adapter / FaceID / DreamBooth-LoRA / DiT in-context。

---

## 机制 3：DreamBooth / Textual Inversion / LoRA ——给画家发一本随身手册

这三个是"训练时介入"派——它们不在推理时给画家递东西，而是**在训练阶段就把"这只猫"塞进画家的脑子里**，让画家以后看到某个特殊词就知道"哦是那只猫"。

### 3a. DreamBooth：把"这只猫"绑到一个稀有词上

**干什么**：拿 3-5 张这只橘猫的照片，**微调整个 t2i 模型**，让模型把一个稀有词（比如 `[V]`，一个生僻 token）和"这只断胡须橘猫"绑死。训练后 prompt 写 `"a photo of [V] cat in a cafe"`，模型就能画出这只猫。

**关键技巧——Class-specific Prior Preservation Loss**：DreamBooth 有个独门绝技。如果只用 5 张橘猫训练，模型会**过拟合**——画啥都画成这只猫。所以 DreamBooth 同时让模型生成一批"普通橘猫"作为参考样本，用一个保留损失防止"这只猫的概念吞掉所有橘猫的概念"。这是 DreamBooth 论文的核心贡献。

### 3b. Textual Inversion：只学一个新单词

**干什么**：不动模型，只**优化一个新的 token embedding**——找到一个新的"伪单词"`<my-cat>`，使得它在翻译官（text encoder）的输出里恰好对应"这只橘猫"。

**优势**：极轻——文件只有几 KB（一个 embedding 向量）；
**劣势**：表达能力有限——模型本身没变，只是多了一个能 trigger 的指针。复杂个体特征经常学不下来。

### 3c. LoRA：在画家脑子里贴一张小补丁

**干什么**：DreamBooth 要更新整个模型（几个 GB），LoRA 只在画家的注意力层里**插入两个小矩阵 A 和 B**（rank=4-32），训练时只动这两个小矩阵。最终的"这只猫"知识被存在这两个矩阵里。

文件大小：通常几十 MB。
训练时间：在 2080 Ti 上几个小时（DreamBooth 全量微调要几天 + 几十 GB 显存）。

**今天的主流配置**：DreamBooth-LoRA——用 DreamBooth 的训练配方（稀有词触发 + prior preservation loss）+ LoRA 的轻量化更新。基本是开源社区做"我家猫的 LoRA"、"我老婆的 LoRA"的标准做法。

### 比喻不准在哪

"教画家学新东西"对 DreamBooth 是准的——**它真的在改画家的脑子**。但要补两点：

- **Textual Inversion 没改画家的脑子**——它只是在翻译官的字典里加了一个新单词。所以它的能力上限低（画家本来不会的事，加一个单词也不会）。
- **LoRA 的"小补丁"是真的小——几十 MB 装一个角色**。这是为什么 LoRA 在角色一致性赛道里赢了：训练成本低、文件小、可以叠加（一个 LoRA 装猫 + 一个 LoRA 装画风 + 一个 LoRA 装动作）。

### 回到橘猫

- 用户拍 5 张这只橘猫不同角度的照片，训一个 DreamBooth-LoRA（半小时），触发词 `[V]`；
- prompt 里都带上 `[V]`，批量生成 10 个场景；
- ✅ **个体特征能保住**——断胡须、缺耳角，因为模型见过这些；
- ✅ 10 张图里的猫**真的是同一只**（不是相似度高，是身份级一致）；
- ⚠️ 训练成本：用户要会训 LoRA、要花时间——产品化时这一步要被工具自动化（即梦的"主体参考"、Lobe2 的"角色"功能本质都在帮用户跑一个零门槛的 LoRA 流程）；
- ⚠️ 训练数据少（只有 1-2 张）时质量大幅下降——DreamBooth 论文给的下限是 3-5 张；
- ⚠️ 多人/多 LoRA 叠加时容易"串味"——两个角色 LoRA 同时启用，可能画出"长着 A 的脸但有 B 的体型"的怪东西。

### 用户视角的失败现象

- **"训出来的猫倒是这只，但每张图都摆同一个姿势"**——训练数据姿势单一，过拟合。
- **"我只想训风格，结果出来的人物也都长一个样"**——训练数据里如果人物特征强，LoRA 会同时学到风格 + 人物特征，分不开。
- **"一张参考图能训 LoRA 吗"**——理论上能（用 1 张图反复增强），但效果远不如 3-5 张。这是为什么单图角色注入是另一条赛道（机制 4）。

---

## 机制 4：PhotoMaker / InstantID / PuLID ——递一张身份证给画家

DreamBooth-LoRA 强在身份保真，弱在要训练。**有没有办法只给一张图、零训练、就把身份注入进去？**

这条赛道叫 **ID embedding / single-image personalization**，专门解人脸（因为人脸有成熟的识别模型可以借力——ArcFace、InsightFace、curricularFace 之类）。

### 4a. PhotoMaker（腾讯 ARC，2023）：把多张参考图叠成一个"堆叠 ID embedding"

- 用 CLIP 视觉编码器抽参考图特征，**把"class token"（"man"/"woman"）和"image embedding"融合**——融合后的向量替换 prompt 里 class token 的位置；
- 多张参考图就把它们的 embedding 在 token 维度上**堆叠**起来（stacked ID embedding），形成一个"统一身份代表"；
- 推理时只要 prompt 里有 `man img` / `woman img` 这种触发词，模型就知道往这个堆叠 embedding 上对齐；
- **单次前向**——不要 fine-tune，几秒出图。

### 4b. InstantID（小红书 InstantX，2024）：人脸 ID + 关键点骨架双控制

- **图像档案员升级**：不用 CLIP，用 InsightFace 的 antelopev2 模型抽专门的人脸 ID embedding（比 CLIP 对身份敏感得多）；
- **两路注入并行**：
  - **Image Adapter**：ID embedding 走 IP-Adapter 式的 decoupled cross-attention（弱身份控制）；
  - **IdentityNet**：一个特殊的 ControlNet——输入是**5 个人脸关键点（两眼、鼻、嘴角两点）+ ID embedding**，用 ID embedding 替代 ControlNet 里的文字条件（强身份控制）；
- 关键点选 5 个而不是密集 OpenPose 关键点——避免太硬把姿态和表情绑死。

### 4c. PuLID（字节豆包，2024）：用对比对齐损失把"身份"和"模型行为"分离

字节自己出的方案，重点解 InstantID 的一个老问题——**身份注入会扰动模型原本的画风/编辑能力**（"我加了 ID 注入，prompt 写的'梵高风格'就不灵了"）。

PuLID 的核心是两个新损失：
- **Contrastive Alignment Loss**：训练时同时跑两条路（有 ID 注入 vs 没 ID 注入），强迫两条路的中间特征**除了人脸区域之外都对齐**——保证 ID 注入只影响脸，不影响画风、构图、光影；
- **Accurate ID Loss**：直接在生成图上跑人脸识别模型，让识别相似度最大化——保证身份真的注入进去了。

PuLID 在论文里展示的效果：身份保真比 InstantID 高，且**插入 ID 前后的画风/光照/背景几乎不变**。

### 比喻不准在哪

"递身份证"听起来像画家拿到一张 ID 卡然后认人——其实**画家只是在每个生成步骤被"反复提醒身份特征向量"**。这个机制**只对人脸成熟**，因为：

- 人脸识别有几十年的研究和成熟模型（ArcFace 等），能给出强语义的 ID 向量；
- 猫、狗、物体没有等价的"识别 ID 向量"——所以 PhotoMaker / InstantID / PuLID 这套方案**对宠物、玩偶、特定物体不直接管用**（要回到 DreamBooth-LoRA 或 IP-Adapter）。

### 回到橘猫

- ❌ **这只断胡须橘猫用不了 PhotoMaker / InstantID / PuLID**——这些方案是为人脸训的，没有"猫脸 ID embedding"对应物；
- ✅ 如果是**用户本人**要批量出图（"把我变成赛博朋克战士"、"把我画成宫崎骏风格"），这套方案是今天的主流——抖音、小红书、即梦的 AI 写真功能基本都是这条路线；
- ⚠️ 即使是人脸，多角度、戴墨镜、侧脸时识别精度会掉，注入的"身份"也会变弱；
- ⚠️ 不同种族、肤色的识别精度有差异——这是底层人脸识别模型自带的偏见，会传递到生成里。

### 用户视角的失败现象

- **"我的脸被改得太像参考图，原本写的'红头发'没出来"**——InstantID 早期常见，PuLID 部分缓解。
- **"侧脸照片注入失败"**——人脸识别在大角度下精度掉。
- **"亚洲人脸输入，出来很西方化的特征"**——底层模型的训练数据偏差。

---

## 机制 5：FLUX Kontext / Step1X-Edit ——让画家边看原图边画新图

这是 DiT 时代的新范式。在 [iteration-precision](../iteration-precision/01-mechanism.md) 里讲过它如何用于"改 A 不动 B"——**同一个机制对"批量保住角色"也是杀手锏**。

### 干什么

把参考图编码成 latent token 序列 $y$，把要生成的图 token 序列 $x$ 拼在后面：`[y₁, y₂, ..., y_N, x₁, x₂, ...]`，**一起进 DiT transformer**——所有 token 之间通过 self-attention 互相看见。模型自动学会"输出图要和输入图保持人物一致"。

为什么这条路在 DiT 时代赢了：DiT 用的是统一的 self-attention 跨所有 token（图像 + 文本一起算），**天然支持"再加一组参考图 token 进来"**——这是 UNet 时代没有的优势。FLUX Kontext 论文专门做了 "character reference" 任务的基准测试，效果显著好过 IP-Adapter 派。

### 多图扩展

架构天然支持多张参考图（多个 $y_1, y_2, \ldots$ 拼接），所以也能：
- **角色 + 风格分离**：参考图 1 = 这只橘猫，参考图 2 = 宫崎骏画风 → 输出"宫崎骏风格的这只猫在咖啡馆"；
- **多角色同框**：参考图 1 = 角色 A，参考图 2 = 角色 B → 输出"A 和 B 一起喝咖啡"。

### 回到橘猫

- 用户给一张断胡须橘猫照片 + prompt "在月球上"；
- ✅ 比 IP-Adapter 强：橘猫的细节（断胡须、缺耳角）保留度更高——DiT 的 self-attention 跨 token 能力让画家能"看见"参考图的更多细节；
- ✅ 比 DreamBooth-LoRA 强：**零训练**，单张图就能用；
- ⚠️ 但仍有微妙的纹理漂移——画家看见的是 latent token 不是原像素；
- ⚠️ 多轮迭代（橘猫 → 加帽子 → 换场景）漂移很小但不为零（Kontext 论文坦承 minimal drift ≠ zero drift）；
- ⚠️ 算力比 IP-Adapter 高——多了一组参考 token，attention 计算量上升。

### 用户视角的失败现象

- **"前两张完美，第五张猫脸已经不是原来那只了"**——多轮漂移累积。
- **"参考图细节复杂时只学到大体，细节漏了"**——latent token 不保像素。
- **"批量出 10 张，互相之间不完全一致"**——每张图独立生成，参考图都是同一张但 seed 不同，输出之间还是有变异。要真正"10 张互相一致"得用机制 6。

---

## 机制 6：StoryDiffusion ——把多张图绑在一起一起画

前面五种机制都解决"输出图 vs 一张参考图"的一致性。**还有一类问题是"一组输出图互相之间要一致"**——典型场景是漫画、连环画、视频分镜：10 张图描述一个故事，每张都是同一个角色在不同场景做不同动作。

### Consistent Self-Attention（一致性自注意力）

StoryDiffusion（南开 + 字节，2024）做的事情很直接：**生成 10 张图时不要分别生成，要并行生成**——每一步去噪时，10 张图的 self-attention K/V **互相共享**。

机制本质和 reference-only 一样（hijack self-attention），但有两个关键差别：
1. **不是一对一参考，是 N 对 N 互相参考**——每张图都能看见其他 9 张；
2. **零训练**——纯推理时的注意力修改，挂在 SD-XL 类模型上即可用。

### Semantic Motion Predictor（语义运动预测器）

StoryDiffusion 还顺手做了视频版：用一致性自注意力先生成一系列关键帧（每张都是同一角色），然后训了一个**语义空间的运动预测器**，把关键帧之间的过渡补成连续视频。比纯 latent 空间的视频插帧更稳定。

### 比喻不准在哪

"10 张图互相参考"听起来像 10 个画家围圈协作——其实**只是把 10 张图的 self-attention 张量在 batch 维度上拼起来一起算**。代价是显存——10 张图同时算注意力，显存压力是单张的 10 倍。所以 StoryDiffusion 的"一次能并行多少张"是有硬上限的，超过就要切窗口。

### 回到橘猫

- 用户给 10 段 prompt（"咖啡馆"、"月球"、"赛博朋克"……），告诉模型"都是同一只橘猫"；
- 一次性并行生成 10 张图，互相通过 consistent self-attention 对齐；
- ✅ 10 张图里的猫角色一致性显著好过"分别生 10 次同一 prompt"——因为它们真的在互相"看"；
- ⚠️ 角色身份是从第一张图里"涌现"出来的——画家自己决定这只猫长什么样，**用户没法精确控制角色长相**（比 DreamBooth-LoRA 弱在这）；
- ⚠️ 显存压力大，实际能一批生几张是工程上限；
- ⚠️ 复杂角色细节（断胡须）依然保不住——consistent self-attention 是对齐"风格和大致角色"，不是 ID 级注入。

### 用户视角的失败现象

- **"10 张里的猫都是一致的，但不是我家那只"**——consistent self-attention 让"批内一致"，但和"参考图一致"是另一回事。要两者都要：在 StoryDiffusion 上叠 IP-Adapter / DreamBooth。
- **"批量超过 8 张就 OOM"**——显存硬上限。
- **"中间某张构图特别奇怪"**——批内 attention 共享会让构图互相"传染"，有时候会出现大家都长得像一个模子里出来的问题。

---

## 视频专属：ConsisID 和"视频角色一致性"的特殊挑战

视频角色一致性多了两个维度：
1. **空间维度**：每一帧里角色要长得一致（同图片）；
2. **时间维度**：跨帧之间不能身份漂移、不能闪烁——前一秒还是这张脸，下一秒就变了不行。

### ConsisID（北大 PKU-YuanGroup，2024，CVPR 2025 Highlight）的解法

ConsisID 是 DiT 视频模型（CogVideoX-5B）上的人脸一致性方案，核心创新是**频率分解**：

- **低频信号注入到浅层**：人脸的整体形状、轮廓、大致结构——和文字 prompt 的 latent 拼接，进 VAE，这是"维持身份的底色"；
- **高频信号注入到注意力块**：
  - **ArcFace 抽 intrinsic 身份特征**（眼睛纹理、嘴唇细节这些抗姿态变化的"身份不变量"）；
  - **CLIP 抽语义特征**（编辑能力、context 理解）；
  - 两路用 Q-Former 融合，对 CLIP 特征做 dropout（避免它干扰身份）；
  - 注入位置：**每个 transformer block 的 Attention 和 FFN 之间**——消融实验证明这位置最优。

### 训练上的两个细节

- **粗到细训练**：先训低频（让模型先抓住"是谁"的大轮廓），再训高频（细节身份特征）。两个分支不竞争。
- **动态 cross-face loss**：训练时偶尔从训练片段**之外**的帧采参考图（同一人但不同时刻），加少量高斯噪声——避免模型"复制粘贴"参考图，提升对未见人脸的泛化。

### 比喻不准在哪

"频率分解"听起来像信号处理——其实它是一个**架构设计哲学**：DiT 不像 UNet 有天然的多分辨率结构（skip connection），所以要**人为把"身份信号"按频率拆成两路注入到不同深度**。这是 v2 比喻里没有的概念——**画家在 DiT 时代的工作方式变了，画室结构本身需要新的解释层**。这是后续要补的。

### 回到橘猫（视频版）

- 用户给一张橘猫照片，要生成一段 5 秒视频"橘猫从沙发跳到窗台"；
- ⚠️ ConsisID **是为人脸训的**，对猫不直接管用——要等等价的"宠物身份注入"工作（开源社区还没看到成熟方案）；
- ✅ 如果是人脸视频生成（"把我变成赛博朋克战士跳舞"），这条路线是今天的开源主流；
- ⚠️ 视频角色一致性的算力成本是图片的 N 倍（N = 帧数）——这是为什么视频里的多 LoRA / 多 ID 注入还远没图片成熟。

### 阿里 Wan 2.1 / 腾讯 HunyuanVideo 在视频角色一致性上的位置

- **Wan 2.1（阿里通义）**：原生支持 image-to-video（首帧条件）、first-last-frame-to-video（首末帧条件）。社区在它上面做了 EchoShot（多镜头同一角色肖像视频）、Phantom（多主体参考）、UniAnimate-DiT（人物动画）等扩展——**多镜头一致性是社区生态在做，不是 Wan 官方核心功能**。
- **VACE**（Wan 子模型）：支持 src_ref_images 多张参考图作为条件——这是机制 5（FLUX Kontext 类）在视频域的对应物。

---

## 把六类机制摆一起

| 机制 | 输入 | 训练成本 | 单图就能用？ | 个体级身份保真 | 风格保真 | 跨样本一致 | 适合什么场景 |
|---|---|---|---|---|---|---|---|
| **IP-Adapter / Plus / FaceID** | 1+ 张参考图 + prompt | 推理零训练（adapter 已训好） | ✅ | ★★ (FaceID 限人脸) | ★★★★ | ★★★ | 风格 / 氛围批量复刻；快速 prototype |
| **Reference-only** | 1 张参考图 + prompt | 零（推理 hack） | ✅ | ★ | ★★★ | ★★ | 零成本风格借鉴；UNet 时代主流，DiT 时代退潮 |
| **DreamBooth-LoRA / Textual Inversion** | 3-5+ 张图训练 | 30 分钟 - 几小时 GPU 训练 | ❌（要训） | ★★★★★ | ★★★★ | ★★★★★ | 长期复用同一角色 / 画风；个人 IP 资产化 |
| **PhotoMaker / InstantID / PuLID** | 1 张人脸照片 + prompt | 推理零训练 | ✅ | ★★★★ (限人脸) | ★★★ | ★★★ | 一键 AI 写真、角色变身；今天产品化最热的赛道 |
| **FLUX Kontext / Step1X-Edit** | 1+ 张参考图 + prompt（DiT 拼接） | 推理零训练 | ✅ | ★★★★ | ★★★★ | ★★★★ | DiT 时代的新基线；多轮编辑 + 角色一致性二合一 |
| **StoryDiffusion (consistent self-attn)** | 一组 prompt | 零（推理 hack） | ✅ | ★★ | ★★★★ | ★★★★★（批内） | 多镜头分镜 / 漫画；不要求和外部参考图绑定时 |
| **ConsisID (视频 ID 注入)** | 1 张人脸照片 + 视频 prompt | 推理零训练 | ✅ | ★★★ (限人脸) | ★★★ | ★★★ | 人脸视频生成；其他主体待业界跟进 |

---

## 失败模式清单（用户视角的统一版）

- **"看着像，但不是这一个"**——CLIP 编码的 IP-Adapter 类方案的固有短板；要个体级一致性必须上 DreamBooth-LoRA / FaceID / Kontext。
- **身份注入挤压了 prompt 的画风/编辑能力**——InstantID 早期常见；PuLID 用对比对齐损失专门解。
- **多张图互相之间一致，但不是参考图里那只**——StoryDiffusion 类批内一致 ≠ 和外部参考一致；要两者都要得叠机制。
- **训 LoRA 训出来"只有一个姿势"**：训练数据姿势单一过拟合；解法是数据多样化 + 加 prior preservation。
- **多 LoRA / 多角色叠加串味**：两个角色 LoRA 同时启用画出"五官混合体"——这是 LoRA 叠加的固有问题，没有完全解。
- **DiT 时代 reference-only / P2P 类老 hack 的退潮**：依赖 UNet self-attention 结构的方法在 FLUX/SD3 上效果减弱，要换新范式（Kontext / Step1X-Edit）。
- **视频里身份漂移 + 闪烁**：单帧能保住的特征跨帧抖动；ConsisID 类方案缓解但远未解决。
- **算力代价随"一致性强度"指数上升**：StoryDiffusion 批量 attention 对显存爆炸；视频 ID 注入对算力指数级吃。

---

## 产品设计师的落地洞察

把上面的内容压缩成几条可以拿去做产品决策的判断：

1. **"风格统一"和"角色一致"是两个不同精度档，产品里必须分开**。普通用户说"批量出图，风格一致就行"——IP-Adapter / Reference-only / consistent self-attn 都够用，零训练秒出。但用户说"这是我家狗，我要它在 10 个场景"——必须上 DreamBooth-LoRA 或 Kontext 类方案，没有捷径。**很多产品把这两档混到一个入口里，是用户挫败感的主因——"我以为传一张图就行，结果出来不是我家那只"**。

2. **"训练 vs 零训练"是真实的产品体验岔路口**。训练派（DreamBooth-LoRA）身份保真天花板高、文件可复用，但用户要承担 10-30 分钟等待和"训练"这个概念的认知负担。零训练派（IP-Adapter / Kontext / FaceID）秒出但保真度有上限。抖音侧用户**不会接受"训练"这个动作**——所以默认路径必须是零训练，把 DreamBooth-LoRA 路径藏到"高级"或"创作者"入口。

3. **人脸是身份注入今天唯一成熟的赛道**。PhotoMaker / InstantID / PuLID 都是人脸方案——这三个工作合起来定义了"AI 写真"今天的产品形态。**对宠物、玩偶、特定物体，今天没有等价的零训练方案**——要么 DreamBooth-LoRA，要么 Kontext（保真度有限）。这是产品功能"覆盖度"的真实边界，不要过度承诺。

4. **DiT 时代的批量一致性正在重新洗牌**。FLUX Kontext / Step1X-Edit 这套"in-context reference"范式在 2025 年快速取代了一部分 IP-Adapter 和 reference-only 的位置——零训练 + 单图 + 多轮稳定，三者兼得。**如果产品里现在还在用 SD 1.5 + IP-Adapter 的老栈，下一代要重新评估技术选型**。

5. **批量风格统一的产品体验关键不是"生成质量"，是"可控的变异度"**。10 张图全一样无聊，全不一样不算"批量"。好的批量产品要能让用户精确控制"哪些维度统一（角色、画风、调性），哪些维度变化（场景、动作、构图）"。机制 5（Kontext 多图）和机制 6（StoryDiffusion）是今天最接近这个体验的两条路。

6. **视频角色一致性是个赛道但还远未成熟**。ConsisID 类方案在人脸上能用，其他主体没工作。Wan 2.1 / HunyuanVideo 的多镜头一致性主要靠社区扩展，不是核心能力。**短期内抖音侧的视频特效更可行的策略是"单镜头精到"，多镜头一致性用 UI 兜底（让用户分镜手动选首帧）**。

---

## 一张图：六类机制怎么解决"批量保住同一个东西"

```
"批量生成同一只断胡须橘猫在 10 个场景"
        │
        ├── ① IP-Adapter（图像档案员压缩参考图，新开一路 cross-attention）
        │      └ 比喻：递一张图给画家当 prompt（CLIP 视觉版本）
        │
        ├── ② Reference-only（助理画师陪画，self-attention K/V 共享）
        │      └ 比喻：画家余光看着助理那张参考画
        │
        ├── ③ DreamBooth / Textual Inversion / LoRA（训练时把"这只猫"装进画家脑子）
        │      └ 比喻：发画家一本随身手册（LoRA 是几十 MB 的小补丁）
        │
        ├── ④ PhotoMaker / InstantID / PuLID（人脸 ID embedding 强注入）
        │      └ 比喻：画家拿到一张人脸身份证，每一笔都对照着画
        │            ⚠️ 限人脸，对猫不管用
        │
        ├── ⑤ FLUX Kontext / Step1X-Edit（参考图 token 拼进 DiT 上下文）
        │      └ 比喻：让画家边看原图边画新图（DiT 时代主流）
        │
        └── ⑥ StoryDiffusion / ConsisID
               ├ a. consistent self-attention：N 张图互相参考一起画
               └ b. ConsisID：视频里的频率分解 ID 注入
```

---

## 还没搞清楚的部分（诚实标记）

- **抖音 / 即梦 / 可灵的"主体参考"功能具体走的哪条路线**：是 IP-Adapter + 自动 prompt 增强？是 PhotoMaker / InstantID 复刻？还是自家训过的 in-context reference 模型？没找到公开文档，要等入职后看内部资料。
- **多 LoRA 同时启用时的"串味"机制**：实践里大家都知道叠两个角色 LoRA 会出怪东西，但 attention 层面到底是什么样的"打架"，论文级研究我没找到——这是个值得查的方向。
- **PhotoMaker / InstantID / PuLID 三家在亚洲面孔上的精度差距**：所有论文用的是西方人为主的基准（CelebA-HQ、FFHQ），亚洲面孔（特别是中老年）的实际效果有公开 benchmark 吗？没找到。
- **视频角色一致性在"非人脸主体"（宠物、玩偶、虚拟角色）上的方案**：开源社区主要做人脸，对其他主体的一致性方案我没找到公开成熟工作——这是产品向的空白区。
- **StoryDiffusion 在 DiT 模型上的可移植性**：论文实验都在 SDXL（UNet）上做。DiT 上做 consistent self-attention 应该能成立（DiT 也有 self-attention），但 batch 维度共享 attention 的显存代价在 DiT 上更恐怖（token 数更多）——没看到工程实测数据。
- **FLUX Kontext 的"character reference"在多角色场景下的具体表现**：单角色 Kontext 论文展示得很好；多角色（A 和 B 同框）我看到的样例不多，需要自己跑实验。

---

## 来源

- **IP-Adapter 论文**（Tencent AI Lab, 2023）：https://huggingface.co/papers/2308.06721 （decoupled cross-attention、22M 参数、与 ControlNet/LoRA 兼容）
- **Diffusers IP-Adapter 文档**：https://huggingface.co/docs/diffusers/en/using-diffusers/ip_adapter （IP-Adapter Plus、FaceID、masking、multi-IP-Adapter、InstantStyle 用法）
- **PhotoMaker 论文**（Tencent ARC, 2023）：https://huggingface.co/papers/2312.04461 （Stacked ID Embedding、单次前向、class token 融合）
- **InstantID 论文**（InstantX / 小红书 / 北大, 2024）：https://huggingface.co/papers/2401.07519 （ID-MLP + IdentityNet、5 个人脸关键点、双路注入）
- **PuLID 论文**（ByteDance, 2024）：https://huggingface.co/papers/2404.16022 （Contrastive Alignment Loss、Accurate ID Loss、Lightning T2I 分支）
- **DreamBooth 论文**（Google Research, 2023）：https://huggingface.co/papers/2208.12242 （稀有 token + class-specific prior preservation loss、3-5 张参考图）
- **Textual Inversion 论文**（Tel Aviv + NVIDIA, 2022）：https://huggingface.co/papers/2208.01618 （单 token embedding 优化、伪单词、几 KB 文件）
- **Diffusers LoRA 训练文档**：https://huggingface.co/docs/diffusers/en/training/lora （rank、target_modules、几十 MB 文件大小、训练成本）
- **FLUX.1 Kontext 模型卡**：https://huggingface.co/black-forest-labs/FLUX.1-Kontext-dev （12B DiT、character/style reference without finetuning、minimal drift）
- **StoryDiffusion 论文**（Nankai + ByteDance, 2024）：https://huggingface.co/papers/2405.01434 （Consistent Self-Attention、Semantic Motion Predictor、零训练）
- **ConsisID 论文**（PKU-YuanGroup, 2024 / CVPR 2025 Highlight）：https://huggingface.co/papers/2411.17440 （频率分解、CogVideoX-5B 基础、ArcFace + CLIP 双塔、Q-Former 融合）
- **ConsisID GitHub**：https://github.com/PKU-YuanGroup/ConsisID （实现细节、49 帧 480×720、44GB → 5-7GB 优化）
- **Wan 2.1 GitHub**（阿里通义，2025）：https://github.com/Wan-Video/Wan2.1 （I2V / FLF2V / VACE 多参考图条件、社区扩展 EchoShot/Phantom 等）
- **Diffusers Reference-only Pipeline 文档**：https://huggingface.co/docs/diffusers/en/api/pipelines/stable_diffusion/stable_diffusion_reference （shared self-attention K/V、style_fidelity、reference_attn / reference_adain）
