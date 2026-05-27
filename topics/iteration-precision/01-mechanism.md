# 迭代修改的精准性：用户改 A 不动 B 是怎么实现的

> 写作说明：本文是 v2 范式的延伸——沿用 [intent-understanding/01-mechanism-v2.md](../intent-understanding/01-mechanism-v2.md) 的"画室五人组"比喻体系，把"迭代修改"这件事讲清楚。如果还没读过 v2，建议先读那篇——本文里"翻译官 / 画家 / 草稿世界 / 骨架图 / 风格手册"这五个角色不再重新解释。

---

## 一句话总结

**让生成模型"改 A 不动 B"，不是让画家更聪明，而是想办法限制画家可以动哪里、参考什么、画多狠。**
工具箱里有六样东西：**禁区图（mask）**、**记忆图（attention map）**、**指令翻译官（instruction model）**、**原图作为参考**、**噪声起点锁定**、**注意力强度旋钮**——每一样都在解决"自由度"问题，自由度越低，迭代越收敛；自由度越高，惊喜越大但也越赌博。

---

## 贯穿案例

整篇文档围绕这一个迭代场景走完全程：

> 用户已经生成了一张图：**"一只戴红色围巾的橘猫坐在咖啡馆窗边看雪"**。
> 现在要改一件事：**"把红围巾换成蓝色，其他都不要动。"**

每一种机制讲完，都会回答一个问题：**这种机制下，红围巾会变成蓝色吗？背景、猫、姿势会保住吗？哪里会失手？**

---

## 为什么 t2i 模型天生不会"改 A 不动 B"

先说清楚问题。在 v2 里我们说过：草稿世界（latent space）是一张地图，prompt 把"目标区域"指出来，seed 决定从哪个起点出发，路径走过去。

**问题来了**：用户原图（红围巾橘猫）对应草稿世界里的某个坐标 P。现在 prompt 改成"蓝围巾橘猫"，目标区域漂移到 P'。看起来只差一个词，但：

- 词变了 → 翻译官输出的语义矩阵变了；
- 语义矩阵变了 → cross-attention 在每个位置的打分全都变了（不只是围巾位置）；
- 打分变了 → 整个去噪轨迹偏离原路径——**所以哪怕 seed 一样、只改一个词，最终图也可能整个变了**（这是 Prompt-to-Prompt 论文里反复展示的现象，原图猫的姿势、背景、光线全都失控）。

> **比喻精确度补丁**：在 v2 里"换 seed 换地图起点 = 不同的图"，给人一种"只要锁住 seed 就能保住"的错觉。**事实是 seed 锁住起点不够——路径还会因为 prompt 微变整条偏掉。** 这就是为什么 t2i 模型直接拿来做编辑会"牵一发动全身"。

接下来六种机制，本质上都是在回答同一个问题：**怎么把"路径"按在原来那条上，只让该改的地方改？**

---

## 机制 1：Inpainting（局部重绘）—— 给画家划禁区

### 干什么

用户在原图上**画一个 mask**：白色区域是要改的（围巾），黑色区域是要保留的（猫、背景）。模型只在 mask 里重新生成内容，mask 外的像素被"按住"。

### 技术怎么做

每一步去噪都做一次"打补丁"：mask 外的区域，用**原图加上当前噪声水平后的版本**强行覆盖回去；mask 内的区域，让画家自由发挥（按 prompt 重新画）。这样画家其实每一步都被偷偷"提醒"了一次背景该长什么样，最后只有 mask 里的内容是新生成的。

更高级的做法（inpainting 专用模型，如 SD-Inpainting、FLUX.1 Fill）：**在 UNet 输入端额外加几个通道**，把 mask 和原图都喂进去训练，让模型从一开始就知道"哪里该画、哪里要接边界"。这种模型生成的过渡更自然——边界不会有明显的拼接痕迹。

### 比喻不准在哪

把 inpainting 当成"画家被拦在白线外"是不准的——更准的说法是：**画家在整张画上都在画，但每画一步就有人把禁区里的部分擦掉换回原样**。所以画家其实"看见"了原图的全貌，他对禁区里有什么东西是知道的——这是自然过渡的来源，也是为什么有时禁区边缘会有微妙变化（画家被允许在边界处协调）。

如果想强制禁区外**完全不变一个像素**，diffusers 提供 `apply_overlay()`，直接把禁区像素硬贴回去——代价是边界更突兀。

### 回到我们的橘猫

- 用户用画笔在围巾区域涂一个 mask，prompt 写 "blue scarf"；
- 模型在 mask 内重画蓝围巾，mask 外的猫、背景、雪景被按住；
- ✅ 围巾变蓝，背景和猫基本不动；
- ⚠️ 围巾的褶皱、披在猫身上的方式可能和原来不一样（mask 内是从噪声重画的，结构不保证）；
- ⚠️ 如果 mask 涂得不干净（碰到了猫的脖子毛），那一小撮毛也会被重画。

### 用户视角的失败现象

- **"改的地方是好的，但边界看着像 PS 痕迹"**——非 inpainting 专用模型容易出，模型不擅长接边。
- **"我只想换颜色，结果围巾的形状也变了"**——mask 区域是从噪声重新生成的，形状不保留。要保形状要叠 ControlNet（见机制 4）。
- **"mask 涂大了，把猫脖子的毛也变了"**——mask 是用户自己定义的"禁区"，模型严格执行。
- **"改完背景颜色微微偏了"**——inpainting 专用模型为了过渡自然，允许 mask 边界外有轻微影响；想完全锁死要用 overlay。

---

## 机制 2：Prompt-to-Prompt（注意力地图劫持）—— 共享画家的记忆

### 干什么

不要 mask，**只改 prompt 的一个词**（"red scarf" → "blue scarf"），但保持图整体结构不变。

### 技术怎么做

回到 v2 第 2 关：cross-attention 是 prompt 影响图片的唯一通道。**画家在画 t 步时，会算出一张"注意力地图" $M_t$——记录"每一块小区域当前最关注哪个 token"。** 这张地图决定了围巾画在哪、猫的轮廓在哪、背景的雪在哪。

P2P 的核心想法：**生成原图时把每一步的 $M_t$ 录下来，生成新图时强行把 $M_t$ 注入回去**——画家被强制按"原来那张图的注意力分布"工作。然后只在被换的那个 token（"red"→"blue"）位置上，用新 token 的 value 替换。

简化算法（来自 P2P 论文）：

1. 用原 prompt + seed 生成原图，记录每一步的 $M_t$；
2. 用新 prompt + 同一个 seed 同步生成；
3. 在前 τ 步注入原图的 $M_t$（保结构），后面几步放手让新词发挥（保 fidelity）；
4. τ 是个超参——τ 大保结构、τ 小保新词被画进去。

P2P 还提供两个变种：
- **Adding a phrase**（加新词，如 "+snow"）：旧 token 用原 attention，新 token 用自己的 attention；
- **Attention re-weighting**（强度旋钮）：把某个词对应的 attention 整体放大或缩小（"more fluffy"，"less snow"）——这就是后来 Midjourney、即梦上"创意度滑块"的原理之一。

### 比喻不准在哪

"共享记忆图"这个比喻让人觉得画家在"参考"原图，但**实际上 $M_t$ 是注入到画家的中间计算里去的，画家根本没有"看到"原图本身**。它只是被强迫按照"画原图时形成的关注分布"来分配每一笔。所以如果原图里围巾是斜披着的、有褶皱，注入 attention 后这些细节会保留——不是因为画家看见了，是因为他的笔触被锁住了。

更要命的一个细节：**P2P 假设原图也是这个模型生成的**——录 attention 是在生成过程里录的。要把它用到"用户上传的真实图片"上，需要先做 inversion（把真实图反推回去成 latent + 模拟 attention 序列），这一步本身就有畸变，是 P2P 在真实图上效果不稳定的根源。

### 回到我们的橘猫

- 原 prompt：`"orange cat with a red scarf in a cafe window"`；
- 新 prompt：`"orange cat with a blue scarf in a cafe window"`，只改一个 token；
- 注入原图的 $M_t$（前 60% 步），让画家按原结构画；
- ✅ 猫姿势、咖啡馆、雪、围巾的位置和褶皱都保留；
- ✅ 围巾从红变蓝，比 inpainting 更"原汁原味"——形状、纹理都接住了；
- ⚠️ 如果改的词需要不同形状（"红围巾"→"红毛衣"），强注入 attention 会让毛衣画成围巾的形状，畸形——这时要把 τ 调小。

### 用户视角的失败现象

- **"我说改 A 它把 B 也改了"**——τ 太小，attention 注入不够，画家自由发挥导致整图重新分布。
- **"形状被锁死了，新东西塞不进来"**——τ 太大，画家被原结构绑死，新词被压制。
- **"对生成图好用，对我自己上传的照片不好用"**——真实图要走 inversion 才能拿到 attention 序列，inversion 本身有损失。

---

## 机制 3：InstructPix2Pix / MagicBrush（指令式编辑模型）—— 教画家听人话

### 干什么

前面两种机制要么要画 mask，要么要懂得"改 prompt 哪个词"。**对普通用户来说太麻烦**。能不能直接说"把围巾改成蓝色"，模型就懂？

### 技术怎么做

InstructPix2Pix（Brooks et al., 2023）做的事情很巧：

1. **造数据**：用 GPT-3 生成大量（原 caption，编辑指令，目标 caption）三元组——比如（"a girl on a horse"，"have her ride a dragon"，"a girl on a dragon"）；
2. **配图**：拿这些 caption 对喂给 Stable Diffusion + Prompt-to-Prompt，**生成"编辑前 / 编辑后"的图片对**（P2P 在这里是数据生产工具，不是推理工具）；
3. **训模型**：在 Stable Diffusion 上加一组输入通道，把"原图"作为 condition、把"指令"作为 prompt，训练一个新的扩散模型，学习"看着原图 + 听指令 → 生成目标图"。

推理时，画家变成了"看图听指令的 editor"——不需要写完整描述，不需要画 mask，只要说 "make the scarf blue"。

InstructPix2Pix 的两个独立 guidance scale 也很关键：
- $s_I$（image guidance）：输出多像原图；
- $s_T$（text guidance）：输出多遵从指令；
- 两个旋钮分别拧——典型用 $s_T \in [5, 10]$，$s_I \in [1.0, 1.5]$。

**MagicBrush** 是后续工作：InstructPix2Pix 训练数据是 GPT-3 + SD 合成的，噪声大、像 SD 出图风。MagicBrush 用真人在 DALL-E 2 上做了 1 万对**手工标注**的真实编辑数据，然后微调 InstructPix2Pix——结果在保留度和编辑准度上都明显提升。这是合成数据 vs 人工数据之争里一个具体案例：合成数据上量快，但精度上限低。

### 比喻不准在哪

把它说成"画家被教会了听人话"是有点过头的——**画家其实没真的"听懂"指令**，它学的是统计映射：见过 X 万张"加了红围巾的猫"和"原始猫"的对比，下次看见类似的猫 + "add red scarf"，它知道往哪里加红色像素。所以遇到训练分布外的指令（"把猫变成它的反面"、"做镜像变换"）就完全失败——InstructPix2Pix 论文自己承认它不会做空间推理（"swap their positions"、"move it left"）和数数。

### 回到我们的橘猫

- 输入原图 + 指令 "change the red scarf to blue"；
- InstructPix2Pix 一步生成新图；
- ✅ 大概率围巾变蓝，猫、背景基本保留——这是它训练数据里见过最多的"颜色替换"任务，强项；
- ⚠️ 整体像素都会被重新生成（不像 inpainting 那样像素级保留），细节会有微小漂移（猫毛纹理、雪花位置）；
- ⚠️ 多轮迭代会累积漂移——改 5 轮后猫已经不是原来那只猫了；
- ⚠️ "把围巾改成披肩，盖住一只前爪"这种结构 + 语义复合编辑，成功率大幅下降——超出训练分布。

### 用户视角的失败现象

- **"指令越复杂越不听话"**——超训练分布。
- **"改了 5 轮发现猫脸都不对了"**——每一轮都是端到端重生成，identity 在迭代中漂移（这是 FLUX Kontext 后来重点解决的问题）。
- **"中文指令效果不如英文"**——训练数据是 GPT-3 英文生成的。

---

## 机制 4：Reference-based / In-context editing（DiT 时代的拼接式编辑）—— 把原图塞进画家眼里

### 干什么

InstructPix2Pix 是 UNet 时代的方案，画家"看"原图是通过额外通道。到 DiT 时代（FLUX、SD3、Step1X-Edit、Sora），出现了一种更优雅的做法：**直接把原图作为视觉 token 序列拼到生成 token 前面，一起进 transformer**——模型在 attention 里同时看原图 token 和生成图 token，自动学会"输出图要参考输入图"。

### 技术怎么做（以 FLUX.1 Kontext 为例）

FLUX.1 Kontext（Black Forest Labs，2025）是这种思路最成功的代表：

1. **架构**：12B 参数的 rectified flow transformer（DiT）；
2. **关键创新**：把原图编码成 latent token 序列 $y$，把生成图 token 序列 $x$ 拼在后面——`[y₁, y₂, ..., y_N, x₁, x₂, ...]`；
3. **位置编码**：用 3D RoPE，给 context token 一个"虚拟 time step"偏移（$y_i$ 在 $t=i$，$x$ 在 $t=0$）——让模型分得清"哪些是参考，哪些是要画的"；
4. **训练**：在 FLUX.1 text-to-image 模型基础上，喂数百万对(原图, 指令, 目标图)继续训练——目标是让模型学会"看着 y 和 c 生成 x"；
5. **多图扩展**：架构天然支持多张原图（多个 $y_1, y_2, ...$ 拼接），所以也能做"参考人物 + 参考风格 → 生成新场景"。

### 比喻不准在哪

把它说成"原图直接塞进画家眼里"是对的，但要补一个细节：**画家看到的不是像素，是 latent token**——经过 VAE 压缩过的高级特征。所以它能感知"这是同一个角色"、"这是同样的光照"，但有时候对像素级的细节（一颗痣、一根头发的走向）还是会丢——这是为什么 Kontext 在 character reference 任务上很强，但绝对像素保真还是不如 inpainting。

**FLUX Kontext 论文专门提的迭代修改卖点（论文原话）**：
> "Robust consistency allows users to refine an image through multiple successive edits with minimal visual drift."

它在论文里专门做了"多轮迭代漂移"基准测试（KontextBench），1024×1024 推理 3-5 秒，character consistency 在多轮编辑下显著好于 GPT-Image-1、Gemini Native Image Gen 等对手——这是**第一次有开源模型把"多轮迭代不漂移"作为头部卖点专门优化**。

类似思路的同期工作：
- **Step1X-Edit**（阶跃星辰，2025）：用 MLLM（Qwen-VL）解析"指令+原图"→ 给 DiT 当条件。本质是更强的指令理解 + 更强的图像编辑分离架构；论文里覆盖 11 种编辑任务，开源；
- **Emu Edit**（Meta，2024）：多任务训练 + 任务 token，让一个模型学会区分"local edit / global edit / segmentation / detection"等任务。

### 回到我们的橘猫

- 输入原图（橘猫红围巾）+ 指令 `"change the scarf to blue"`；
- 模型把原图 token 和生成图 token 一起进 transformer；
- ✅ 围巾变蓝；
- ✅ 猫的脸、姿势、咖啡馆背景都保留得相当好——这是 Kontext 论文里展示的核心能力；
- ✅ 再来一轮 `"now also add a snowflake hat"`——再下一轮 `"make the cat sleepy"`——多轮下来漂移很小；
- ⚠️ 但仍有微妙的纹理漂移——画家看见的是 latent token 不是原像素；
- ⚠️ 论文坦承多轮后还是会漂移（minimal ≠ zero），只是比对手少。

### 用户视角的失败现象

- **"前两轮编辑很完美，第五轮发现猫已经不是原来那只了"**——多轮漂移，目前所有方法的共同短板，Kontext 改善但没消除。
- **"换衣服的同时人脸也微微变了"**——latent token 不保像素。
- **"指令复杂时只听了前半句"**——MLLM/text encoder 仍是瓶颈，复杂复合指令会丢。

---

## 机制 5：Seed 锁定 + low-strength img2img（最朴素的工程兜底）—— 不动地图起点，只走半路

### 干什么

不引入新模型，纯靠 t2i 模型的两个旋钮做"轻度编辑"。

### 技术怎么做

img2img 流程：把原图编码到 latent → 加噪声到第 t 步（不是从纯噪声开始）→ 用新 prompt 去噪走完剩下的路 → 得到新图。

两个关键参数：
- **strength**（也叫 denoising strength）：加多少步噪声。strength=0.3 意味着只加 30% 的噪声，去噪 30 步——画家只在原图基础上"小修"。strength=1.0 等于完全重画；
- **seed**：去噪过程的随机数种子，相同 seed 走同样的随机扰动路径。

把 strength 调低（0.2-0.4），新 prompt 改一个词——理论上画家从一个"接近原图的 latent"出发，按新 prompt 走一小段。

### 比喻不准在哪

听起来像"画家在原图上轻轻刷一层"，但**实际上画家是从一个被加噪到看不出原貌的中间态出发，按新 prompt 重画**。strength 只决定"重画的幅度"，不决定"重画在哪里"——所以**整张图都在变，只是变化幅度小**。这跟 inpainting 的"硬禁区"是本质区别。

### 回到我们的橘猫

- strength=0.3，seed 锁定，prompt 改成 "blue scarf"；
- ✅ 整体保留度高，看起来像"原图但围巾偏蓝了"；
- ⚠️ 但整张图都在微变——背景的雪粒、猫毛的纹理、咖啡馆的光线都和原图不完全一致；
- ⚠️ strength 太低（<0.2）：新词进不来，围巾还是红的；
- ⚠️ strength 太高（>0.5）：开始整张图重画，猫姿势可能变；
- ⚠️ 调参靠人工试——不同 prompt、不同图片的最优 strength 不一样。

### 用户视角的失败现象

- **"整张图都微微变了，没有一处是真的没动"**——img2img 的本质，全图重生成。
- **"调 strength 靠玄学"**——没有自动化方法。
- **"在新模型（FLUX、SD3）上效果反而差"**——这些模型为高质量输出训练的，对低 strength 的中间态 latent 反应不一致，img2img 在 DiT 类模型上没在 UNet 时代好用。

这是 t2i 模型的"工程兜底"，便宜（不要新模型），但精度低。**今天主流编辑方案都不靠它**——Inpainting、Kontext、IP2P 都比它精准。

---

## 机制 6：视频迭代修改（多了一个时间维度的烫手山芋）

### 干什么

视频版的 inpainting / 编辑要解决的问题是：**改 A 不动 B**（空间维度），**还要改完每一帧之间不闪烁**（时间维度）。

### 技术怎么做

主要思路两种：

**a. 视频 inpainting（mask 跨时间传播）**
用户在第一帧上画 mask，模型自动把 mask 传播到后续帧（追踪 mask 区域的运动），然后在每一帧的 mask 区域里重生成内容。Runway 的 Inpainting、可灵的"局部重绘"、即梦的"擦除替换"基本都是这个思路。

技术挑战：mask 要跟着物体动（猫转头时围巾的 mask 要跟着转），靠的是 SAM-2、视频分割模型这类外部追踪器。重生成时还要保证帧间一致——通常用 video diffusion 模型，去噪时 spacetime patch 同时覆盖空间 + 时间，让相邻帧不闪烁。

**b. 关键帧编辑 + 整体重生**
用户编辑首帧（用图像 inpainting 工具改成"蓝围巾"），然后用 video-to-video 模型（如 Runway Gen-3 turbo、Luma Dream Machine）把改过的首帧作为 condition 重新生成整段视频。

**Sora 的"Remix"功能**（OpenAI Sora）属于这一类——用户用自然语言描述要改什么（"change the scarf to blue"），Sora 会基于原视频重新生成。但 OpenAI 的官方说法里 Sora 编辑能力公开信息有限，具体技术细节没有完整论文披露——**这部分我不能给确定结论，标"待验证"**。

### 比喻不准在哪

把视频编辑想成"把图像 inpainting 一帧一帧做"是错的——逐帧独立做会**严重闪烁**（每帧 mask 内的内容都是独立采样的，颜色和纹理在帧间跳变）。**真正的视频编辑必须在时间维度上联合去噪**——这就是为什么视频编辑在算力上比图片编辑贵几个数量级。

### 回到我们的橘猫（假设是个 5 秒视频）

- a 路线：第一帧画围巾 mask，模型追踪猫脖子运动，每一帧都在追踪到的区域生成蓝围巾；
- b 路线：把第一帧改成蓝围巾，整段重生；
- ✅ 围巾大概率变蓝；
- ⚠️ 闪烁是常见失败——围巾的蓝色在不同帧明度不一样；
- ⚠️ 追踪掉点——猫快速转头时 mask 跟丢，丢的那几帧围巾又变成红的或畸形；
- ⚠️ 重生时画面整体细节会变（背景的雪、咖啡馆里的人物）。

### 用户视角的失败现象

- **"前两秒围巾是蓝的，第三秒突然闪一下变回红"**——mask 追踪丢了。
- **"局部重绘以后，背景里的人物变成另一个人"**——视频重生成不像图像编辑那样精细，整段都被部分重画。
- **"5 秒视频改 3 秒"——算力极贵**：视频生成本身就贵，加上编辑要重新去噪，多轮迭代用户基本忍不了等待。

---

## 把六种机制摆一起

| 机制 | 输入 | 改动方式 | 改 A 不动 B 强度 | 多轮迭代友好度 | 用户操作复杂度 |
|---|---|---|---|---|---|
| **Inpainting** | 原图 + mask + prompt | mask 内重画 | ★★★★★（mask 外像素硬保） | ★★★（每轮要画 mask） | ★★（要画 mask） |
| **Prompt-to-Prompt** | 原 prompt + 新 prompt | attention 注入 | ★★★★（原图必须是模型生成的） | ★★（attention 序列管理麻烦） | ★（只改一个词） |
| **InstructPix2Pix / MagicBrush** | 原图 + 指令 | 端到端重生 | ★★★ | ★★（多轮漂移明显） | ★（说人话） |
| **FLUX Kontext / Step1X-Edit** | 原图 + 指令（DiT 拼接） | 上下文驱动重生 | ★★★★（character 一致性好） | ★★★★（论文专门优化） | ★（说人话） |
| **Seed + low-strength img2img** | 原图 + 新 prompt | 半程去噪 | ★★（全图微变） | ★★（漂移明显） | ★ |
| **视频 inpainting / Remix** | 原视频 + mask 或指令 | mask 传播或重生 | ★★（闪烁、追踪掉点） | ★（算力 + 漂移双重压力） | ★★（视频更难精确表达） |

---

## 信息丢失点 / 失败模式（用户视角的统一清单）

- **多轮漂移**：所有"端到端重生"类方法（IP2P、Kontext、img2img）都有——每一轮的输出会被下一轮当输入，误差累积。Kontext 论文专门提"minimal drift"作为卖点，反过来说明这是行业共识难题。
- **Mask 边界破绽**：非 inpainting 专用模型容易留拼接痕迹；inpainting 专用模型为自然过渡允许微变 → 想完全锁死要 overlay。
- **Identity 漂移**：换衣服的时候人脸也变了——latent token 不保像素，DiT 类编辑器普遍存在。
- **指令理解的最后一公里**："把猫变成它的反面"、"交换这两个东西的位置"——所有方法在这类需要空间 / 因果推理的指令上都翻车（IP2P 论文坦承）。
- **真实图 vs 生成图的鸿沟**：P2P 在生成图上效果好，在用户上传的真实图上要走 inversion，效果打折。Kontext 这种 in-context 方法天然支持真实图，是行业方向。
- **视频的双重诅咒**：空间精度 + 时间一致性 + 算力，三选一都难，全要更难。

---

## 产品设计师的落地洞察

1. **"改 A 不动 B"是产品体验里被严重低估的事情**。一次出图惊艳容易，五轮迭代不崩盘很难。在抖音特效场景里，普通用户的修改路径是"先出一个差不多的，再小调"——多轮漂移直接决定"用户最后能不能拿到一个真的能发出去的视频"。**这是用户主路径上的核心体验瓶颈，不是 nice-to-have**。

2. **"用户要不要画 mask"是真实的产品岔路口**。画 mask 精度最高但门槛最高（手机上画 mask 是反人性的）；说人话最舒服但精度低。一种已经被验证的中间方案：**让 MLLM 把"指令"自动转换成 mask**（"把围巾涂掉"→ 模型自动分割出围巾区域 → 走 inpainting）——SAM-2 + Grounded-SAM 让这个变得可行。抖音侧应该把这条路径作为默认体验，专业用户再开放手画。

3. **"端到端重生 vs 区域局部"是不同精度档**。普通用户要"换颜色 / 换风格 / 加东西"——这种 Kontext / IP2P 类的端到端足够；要"P 掉某个东西"、"换某一小块"——端到端会污染太多区域，必须走 inpainting。产品里两条路要并存，不要妄图一个方案搞定所有。

4. **多轮迭代的"漂移问题"应该用 UI 兜底，不要赌模型完全不漂**。比如让用户能随时回到任意一轮的"基准图"再分支编辑，而不是只能线性累加；比如自动保留每一轮的 seed 和参数，方便定位是哪一轮开始崩的。**模型层做不到的事，产品层用历史记录和分支结构兜底**。

5. **视频迭代在今天的算力下是个尚未解决的问题**。给用户开放"局部重绘"功能时要慎重——抖音侧用户不会接受 30 秒的等待；同时多轮迭代的视频编辑成本会指数上升。短期内更可行的是**"一键变体"（同一个意图换几个 seed 出几个版本，让用户挑）**而不是"多轮微调"。这是个产品策略选择，不是技术选择。

6. **DiT 时代的编辑是新的产品基线**。FLUX Kontext / Step1X-Edit 这套"原图 token 拼接 + MLLM 指令解析 + DiT 生成"的范式在 2025 年快速成熟。如果抖音特效团队的下一代图像编辑还在用 InstructPix2Pix / SDEdit 时代的方案，产品体验会和外部产品（GPT-Image、Gemini Native Image）拉开越来越大的差距。**这是技术路线选型决策**。

---

## 一张图：六种机制怎么解决"改 A 不动 B"

```
"把红围巾改成蓝色"
        │
        ├── ① Inpainting：用户画禁区，禁区外像素硬保留
        │      └ 比喻：给画家递一张"不准画的区域"地图
        │
        ├── ② Prompt-to-Prompt：录原图的注意力地图，新图强行注入
        │      └ 比喻：把画家画原图时的笔触分布按住不动
        │
        ├── ③ InstructPix2Pix / MagicBrush：训练一个"听指令的画家"
        │      └ 比喻：教画家学会成千上万种"X→Y"的对应
        │
        ├── ④ FLUX Kontext / Step1X-Edit：原图 token 直接拼进生成上下文
        │      └ 比喻：让画家边看原图边画新图（DiT 时代主流）
        │
        ├── ⑤ Seed + low-strength img2img：从原图加噪后的中间态出发
        │      └ 比喻：让画家从原图变模糊的版本继续画几笔
        │
        └── ⑥ 视频 inpainting / Remix：mask 跨时间传播或整段重生
               └ 比喻：①或④在每一帧上做，还要保证帧间不闪
```

---

## 还没搞清楚的部分（诚实标记）

- **Sora 的 Remix / Re-cut / Blend / Loop 具体技术实现**：OpenAI 公开技术信息有限，help center 文档我没拿到。猜测 Remix 是关键帧改 + 整段重生，但**没有论文级证据，标待验证**。
- **可灵和即梦的"局部重绘"具体走的是哪条路线**：是 mask 跨帧传播 + 视频 inpainting，还是关键帧改 + v2v？没找到公开技术文档，要等入职后看内部资料。
- **DiT 模型上的 Prompt-to-Prompt 是否还成立**：P2P 是基于 UNet + cross-attention 的，DiT（FLUX、SD3）attention 模式不同（self-attention 跨 image+text token），P2P 注入的对象变了。**DiT 时代的"attention 注入"研究公开材料较少**——这是个值得查的方向。
- **Inpainting 在 DiT 模型上（FLUX.1 Fill、SD3 Inpainting）相比 UNet 时代的具体差距**：边界自然度、对原图的保留度有量化对比吗？没找到完整 benchmark。
- **多轮迭代漂移的量化指标**：Kontext 论文用 KontextBench 测了，但开源社区还没有公认的"多轮编辑一致性"标准 benchmark。这是个产品评估上的盲点。

---

## 来源

- **Diffusers Inpainting 文档**：https://huggingface.co/docs/diffusers/en/using-diffusers/inpaint （mask 的工作机制、strength、padding_mask_crop、apply_overlay）
- **Diffusers img2img 文档**：https://huggingface.co/docs/diffusers/en/using-diffusers/img2img （strength 参数、与 inpainting 的差异）
- **Prompt-to-Prompt 论文**（Hertz et al., Google Research, 2022）：https://arxiv.org/abs/2208.01626 （cross-attention map 注入、word swap / adding phrase / re-weighting 三种操作、τ 控制）
- **InstructPix2Pix 论文**（Brooks, Holynski, Efros, Berkeley, 2023）：https://arxiv.org/abs/2211.09800 （GPT-3 + SD + P2P 合成数据、双 guidance scale、空间推理失败）
- **MagicBrush 论文**（Zhang et al., OSU, 2023）：https://arxiv.org/abs/2306.10012 （手工标注 1 万对，微调 IP2P 显著提升）
- **FLUX.1 Kontext 论文**（Black Forest Labs, 2025）：https://arxiv.org/abs/2506.15742 （DiT + token concat、3D RoPE 虚拟时间步、KontextBench、多轮迭代 minimal drift）
- **FLUX.1 Kontext 模型卡**：https://huggingface.co/black-forest-labs/FLUX.1-Kontext-dev （12B 参数、guidance distillation、character/style reference）
- **Step1X-Edit 论文**（阶跃星辰, 2025）：https://arxiv.org/abs/2504.17761 （Qwen-VL + DiT、token concat、11 类编辑任务、GEdit-Bench）
- **FLUX.1 Fill**（Black Forest Labs）：https://huggingface.co/black-forest-labs/FLUX.1-Fill-dev （DiT 时代 inpainting 模型）
