# 意图理解：用户输入是怎么变成生成结果的

> 给产品设计师的技术链路解释。读完应该能：1）听懂行业讨论里的术语；2）做产品决策时知道每一步在干嘛；3）出问题时能定位到是哪一环出的。

---

## 一句话总结

**"意图理解"不是一步，是一条流水线。** 用户的文字、图片、参考资料，先被各自的"翻译官"转成模型能读懂的数字向量（embedding），然后在生成过程中通过一种叫"cross-attention"的机制，被反复"喂"给一个负责画画的网络。每一步都会有信息损耗——所以同样一句 prompt，不同模型、不同环节出问题，结果会差很多。

---

## 整体链路图

```
┌────────────────────────────────────────────────────────────────────────┐
│  用户输入                                                               │
│    [文字 prompt]   [参考图]    [结构控制]   [LoRA/微调]                 │
└──────┬──────────────┬───────────┬──────────────┬─────────────────────┘
       ↓              ↓           ↓              ↓
   ┌───────┐    ┌─────────┐  ┌─────────┐   ┌──────────┐
   │ 文本   │    │ 图像     │  │ ControlNet│  │ LoRA/    │
   │ 编码器  │    │ 编码器   │  │ 编码器    │  │ 微调权重  │
   │CLIP/T5│    │CLIP-V等 │  │(canny等) │  │          │
   └───┬───┘    └────┬────┘  └────┬─────┘   └────┬─────┘
       │ 文本向量      │ 图像向量    │ 结构特征      │ 风格/概念偏置
       ↓              ↓           ↓              ↓
   ┌──────────────────────────────────────────────────────┐
   │              生成主网络（UNet 或 DiT）                 │
   │    通过 cross-attention 把上述所有条件融进去            │
   │    从一团随机噪声开始，反复"去噪"出图                    │
   └──────────────────────┬───────────────────────────────┘
                          ↓
                   ┌──────────────┐
                   │   VAE 解码器   │
                   │  把"压缩态"     │
                   │  还原成像素     │
                   └──────┬───────┘
                          ↓
                     最终图像/视频
```

**两个关键概念先说清楚：**

- **Embedding（向量）**：把任何输入（文字、图片）转成一串数字。这串数字才是模型能"读"的语言。
- **Latent space（潜空间）**：模型不直接画像素，而是在一个被压缩过的"草稿空间"里画。512×512 的图在潜空间里可能只有 64×64。生成完了再用解码器还原成像素。这是为什么 Stable Diffusion 能跑在普通显卡上的原因之一。

---

## 1. 文本处理链路（生图模型）

### 1.1 文本编码器（Text Encoder）：把文字翻译成向量

文本编码器是一个独立的语言模型，专门干一件事：**把 prompt 切成 token，再把每个 token 映射成一串数字**。

主流模型用什么编码器，差别巨大：

| 模型 | 用什么文本编码器 | 特点 |
|---|---|---|
| Stable Diffusion 1.5 | CLIP ViT-L/14 | 基础款。一句话只能装 77 个 token，"长 prompt 后面被截断" 的根源 |
| SDXL | CLIP-L + CLIP-G（两个） | 双编码器，一个抓概念、一个抓细节 |
| Stable Diffusion 3 | 2 个 CLIP + 1 个 T5（4.7B） | T5 让模型能正确写出图里的英文字 |
| FLUX.1 | CLIP-L + T5-XXL | 长 prompt、复杂场景理解明显更强 |
| HunyuanVideo（腾讯） | 多模态 LLM（MLLM）+ 双向 token refiner | 用 LLM 当编码器，理解能力更强但是要补一个组件 |
| Wan 2.1（阿里） | umT5-xxl | 多语言，中文 prompt 优势 |

**为什么是 CLIP？**
CLIP 是 OpenAI 训练的一个对比学习模型，它同时训练了一个文字编码器和一个图片编码器，让"匹配的文字-图片对"在向量空间里靠近。所以 CLIP 的文字编码器天生就"懂图"，输出的向量本身就和视觉概念对齐。

**为什么再加一个 T5？**
CLIP 是为"匹配图文"训练的，对**精确的文本细节**（比如 "三个红色的苹果在桌子左边"）理解有限。T5 是 Google 的纯文本大模型，处理复杂语义的能力强。SD3 和 FLUX 把两者拼起来：CLIP 抓视觉相关性，T5 抓精细描述。

**为什么字节、腾讯、阿里在换 LLM 当编码器？**
传统 CLIP/T5 都是几年前的模型，对复杂指令、多步推理力不从心。MLLM（比如 HunyuanVideo 用的）本身是多模态大模型，理解能力更强，能识别更细的语义层次。但代价是要加额外的组件来弥补 LLM 的"单向注意力"短板。

### 1.2 编码后的"文本特征"长什么样？

不是一个数字，是**一组向量**。每个 token 一行，整个 prompt 是一个矩阵。比如 SD 1.5 输出的是 (77, 768)：77 个 token 槽位，每个槽位 768 维。这个矩阵就是后面 cross-attention 反复来取的"参考资料"。

### 1.3 Cross-Attention：决定"prompt 里的某个词落在图片哪个区域"

**这是整条链路最核心、最容易混淆、也最值得记住的机制。**

生成主网络（UNet 或 DiT）在去噪过程中，每一层都会做这件事：

1. 把当前的"半成品图像"切成很多小块（每块对应图片的一个区域）；
2. 让每个小块去**查询**文本向量矩阵——"我这块该听哪个词的？"；
3. 每个小块根据查询结果，按权重把多个 token 的特征融进自己。

举个例子：prompt 是 "a cat sitting on a sofa"。生成到中期时：
- 图片中央的某些块会**强烈关注 "cat"** 这个 token，往猫的特征靠；
- 周围的块会**强烈关注 "sofa"**，往沙发的特征靠；
- "sitting" 这个动作词会被多个区域共同关注，影响整体姿态。

**Cross-attention 是 prompt 影响图片的唯一通道**。如果某个词在 cross-attention 里没被某个区域关注到，那这个词在最终结果里就**没声音**。这就是"我说红苹果它没给我红"的常见根源。

### 1.4 为什么同一个 prompt 给不同模型，结果差很多？

根本原因有四层（按影响力排序）：

1. **训练数据不同**：模型只能生成它见过的东西。"赛博朋克"在 SD 1.5 里和在 Midjourney 里指代的视觉概念可能完全不同。
2. **文本编码器不同**：CLIP 对 "在...左边" 这种空间关系几乎不敏感，T5 好一些，MLLM 最好。换了编码器，等于换了"翻译官"。
3. **Cross-attention 实现不同**：SDXL 的 attention 层数和 SD3 的 MM-DiT 不是一回事，同一个向量进去，"听话程度"不同。
4. **VAE 不同**：解码器的"画风"会带来最后一道偏色/锐度的差别。

**给产品的判断：** 当用户抱怨"换了模型出图变了"，多半是 1+2 的复合问题，不是简单的"调一下参数能解决"。

---

## 2. 非文本输入的处理链路

文字 prompt 是模型最早学会的输入。后来出现的图片输入、结构控制、风格参考，本质上都是**额外的"侧线"**——它们不替换 prompt，而是在 cross-attention 的旁边再开一条通道。

### 2.1 img2img：把参考图当"起点"

**机制**：
1. 用 VAE 编码器把参考图压缩成潜空间向量；
2. 往这个向量里加噪声（加多少由 `strength` 参数决定，0.4 = 加 40% 的噪，0.8 = 加 80% 的噪）；
3. 从这个"半噪态"开始去噪，而不是从纯噪声开始。

`strength` 越低，越像原图；越高，越像 prompt 描述的新图。`strength=1.0` 就退化成普通的文生图（参考图被噪声完全淹没）。

**关键认知**：img2img 不是"理解参考图的内容"，它只是"借参考图的形状当种子"。你给一张猫的照片，写 prompt "a dog"，结果可能是一只长得像那只猫的狗——构图、姿态、色调像猫，主体变成了狗。

### 2.2 ControlNet：精确控制构图、姿态、轮廓

**它是什么**：一个**额外训练的小网络**，专门接收"结构信号"（线稿/深度图/姿态骨架/语义分割图等），把这些信号注入主网络。

**机制**：
- 主网络（UNet）的权重被冻结；
- ControlNet 复制主网络的一部分结构作为副本；
- 副本接收结构图（比如一张 canny 线稿），处理后用"零卷积"层把信息加回主网络的对应层；
- 主网络还在听 prompt，但同时被结构图"约束"。

**和 prompt 的关系**：ControlNet 决定**形状/构图/骨架**，prompt 决定**内容/材质/风格**。两个一起用：
- prompt：金毛犬，工作室灯光，4k
- ControlNet（pose）：人物舞蹈姿势的骨架图
- 结果：一只摆出舞蹈姿势的金毛犬

**`controlnet_conditioning_scale`** 参数控制结构的"强度"。0.5 是软约束，1.0 是强约束，1.0+ 容易把图像"绑死"导致勉强出图。

**给产品的判断**：用户说"我要那个东西的形状但内容换掉"——这是 ControlNet。说"我要那个东西的样子但摆个不同姿势"——这是 ControlNet（pose）+ 文字 prompt。

### 2.3 IP-Adapter：让"参考图的风格/主体"传递到生成结果

**它是什么**：一个**轻量适配器**，专门把图像当成 prompt 用。

**机制**：
- 用 CLIP 的图像编码器，把参考图压缩成图像向量；
- 主网络的 cross-attention 旁边**新加一组**专门给图像向量用的 attention 层（叫"解耦 cross-attention"）；
- 文字 cross-attention 还在管"语义"，新加的图像 cross-attention 管"参考图的味道"。

**和 ControlNet 的核心差别**：
- ControlNet 控制**结构**（形状、轮廓、姿态）；
- IP-Adapter 控制**风格/特征**（画风、材质、人脸特征、整体氛围）；
- 两个可以同时用：ControlNet 锁结构，IP-Adapter 锁风格。

**`ip_adapter_scale`** 参数：0 = 完全忽略参考图，1.0 = 主要听参考图。0.5-0.8 是常见甜点区。

**特殊变体**：
- IP-Adapter FaceID：用人脸识别专用 embedding，更准；
- IP-Adapter Plus：用更细粒度的 patch embedding，能保留更多细节。

**给产品的判断**：用户说"我要这个画风/这个人物/这个氛围"——IP-Adapter。说"我要这个东西本身"——img2img + low strength 或 IP-Adapter Plus。

### 2.4 LoRA：让模型"额外学会一个东西"

**它是什么**：一个**几 MB 的小补丁**，专门微调主网络（特别是 cross-attention 层）的一小部分权重。

**机制**：
- 主网络权重冻结；
- 在 cross-attention 层旁边注入一组小矩阵（low-rank matrices）；
- 训练时只训这组小矩阵，让它学会某个特定的概念/风格/人物；
- 推理时把小矩阵的影响"加"到主网络上。

**和 prompt 的协作**：
- LoRA 不替代 prompt，它**改变模型对某些词的理解**。
- 训练时通常会绑定一个**触发词**（trigger word），比如训练一个"赛博朋克猫"的 LoRA，触发词是 "cybercat"。
- 推理时 prompt 里带上 "cybercat"，LoRA 才会被激活到最强。

**和 ControlNet/IP-Adapter 的差别**：
| 维度 | LoRA | ControlNet | IP-Adapter |
|---|---|---|---|
| 改的是 | 模型本身的认知 | 加一个并行结构网络 | 加一个并行图像通道 |
| 输入 | 仅文字 prompt | 文字 + 结构图 | 文字 + 参考图 |
| 解决什么 | "教模型不认识的东西" | "锁住构图/形状" | "锁住风格/主体外观" |
| 文件大小 | ~3-200MB | 几百 MB | 几十 MB |
| 是否需要训练 | 是（每个概念一次） | 不需要（按类型预训练好了） | 不需要 |

**关键直觉**：
- **LoRA 改"模型记忆"**——教它新东西；
- **ControlNet 改"画的过程"**——告诉它怎么画；
- **IP-Adapter 改"参考资料"**——给它看一张图。

三个可以叠加用，业内做"角色一致性 + 特定姿势 + 特定风格"经常一次性用全。

---

## 3. 视频生成的特殊性

视频生成**底层逻辑和图片一样**——都是"从噪声去噪到结果"。但因为多了"时间"这个维度，意图理解链路有几个明显升级。

### 3.1 关键差别：从 patch 到 spacetime patch

图片生成把图切成 patch（小方块），每个 patch 是空间维度的一个 token。视频生成把视频切成 **spacetime patch**——一个 token 同时覆盖一小块画面 × 一小段时间。

这意味着 cross-attention 在工作时，每个 spacetime patch 既要听 prompt 里"画什么"的部分，又要听"动什么"的部分。"a cat **jumping** off a sofa"——"jumping" 这个词必须能被时间维度上多个连续 patch 共同关注，否则猫就只是站着不动。

### 3.2 文本编码器升级 + 二次撰写（recaptioning）

**Sora、HunyuanVideo、Wan 普遍做了一件关键的事：训练数据的 caption 全部用一个"描述器模型"重写一遍**。

为什么？
- 互联网上视频的 alt text、标题都是垃圾，根本不描述画面；
- 用户写的 prompt 又通常很短（"一只猫在跳"）；
- 训练时模型见到的是"详细到 200 字的精细描述"，推理时用户给的是"5 字短句"——分布对不上，意图理解就崩。

**主流做法**：
- **训练时**：用一个 video-to-text 模型（比如 VideoCoCa）给每条训练视频写详细 caption；
- **推理时**：用一个 LLM（比如 Sora 用 GPT-4V，HunyuanVideo 有专门的 PromptRewrite 模型，Wan 用 Qwen）把用户的短 prompt **扩写成详细描述**，然后再喂给主模型。

**这就是为什么 Sora、可灵的"prompt 优化"按钮存在**——它不是产品贴心，是模型必须的一道工序。

**给产品的判断**：当用户抱怨"我写得很清楚但出来不对"，要看他的 prompt 是否被默默改写了。被改写后的 prompt 才是模型真正看到的。这是一道**信息黑盒**——产品体验上要不要把改写后的内容暴露给用户，是个真实的设计抉择。

### 3.3 时序一致性是怎么"从文字里拿出来的"

简短答案：**主要不是从文字里拿出来的，是从训练数据里学出来的**。

文字只能给"语义提示"（"一个人在跑步"），但具体跑步的连贯姿态、布料随风摆动的物理、镜头跟随的运镜——这些从 prompt 里挖不出来，模型必须**在大量真实视频里见过类似的运动模式**才能复现。

这就解释了几个常见现象：
- **物理违和**：模型没见过/见过不够多类似物理场景；
- **动作不连贯**：训练数据里这个动作覆盖度不够；
- **镜头运动僵硬**：缺少多机位、多镜头切换的训练样本。

OpenAI 自己也承认 Sora 在"复杂物理"、"因果关系"、"分清左右"上有明显短板——这些都是**时间维度上的"意图理解"**还没完全解决的问题。

### 3.4 主流视频模型在意图理解上的差异（粗略对比）

| 模型 | 文本编码 | 推理时 prompt 处理 | 主架构 | 特点 |
|---|---|---|---|---|
| Sora（OpenAI） | 类 CLIP + LLM 扩写 | GPT-4V 自动扩写 | DiT + spacetime patch | 1 分钟长视频、运镜与连贯性强 |
| Runway Gen-3 | 内部模型 | 训练用了"时间密集型 caption" | 多模态训练 | 电影感/运镜术语理解好 |
| 可灵 KLING（快手） | 公开信息少 | 内部优化 | DiT 类 | 物理表现强、长镜头稳定 |
| HunyuanVideo（腾讯） | MLLM + bidirectional refiner | 专门的 PromptRewrite 模型 | 双流-单流混合 DiT | 开源、对复杂指令理解强 |
| Wan 2.1（阿里） | umT5-xxl（多语言） | Qwen 扩写 | DiT + Flow Matching | 中英文 prompt 都强、能在视频里写文字 |

**给产品的判断**：选模型时，**意图理解强 ≠ 画质高**。强意图理解意味着 prompt 里的细节（比如"左边的人转头看右边"）更可能被实现；高画质意味着每一帧的视觉质感好。这是两个独立维度。

---

## 4. 出问题的话，问题会出在哪一环

按"用户最常抱怨的现象 → 根因在哪"梳理：

### "我说 A，它给我 B"——语义偏离

可能根源（按概率排）：
1. **文本编码器没听懂细节**。CLIP 对空间关系（"在左边/右边/后面"）几乎无感；对数量（"三个"vs"四个"）也常常失败。换 T5/MLLM 系的模型会好很多。
2. **训练数据里这个概念稀缺**。SD 1.5 几乎不会画"穿汉服的赛博朋克少女"，因为它没见过足够多的汉服+赛博朋克的组合样本。
3. **Cross-attention 没把这个词路由到正确区域**。这是 prompt engineering 出现的根本原因——通过调整词序、加权重、加修饰词，让 attention 更稳定地"听到"关键概念。
4. **CFG（classifier-free guidance）数值不当**。CFG 太低，模型自由发挥忽视 prompt；CFG 太高，过度放大某些特征导致畸变。
5. **视频/复杂模型的 prompt rewrite 改坏了**。用户的精确意图被 rewrite 成另一个意思后，模型看到的根本不是用户想要的。

### "多轮调整不收敛"——越调越崩

可能根源：
1. **每一轮的 seed 在变**。同一个 prompt + 同一个 seed 应该出同一张图。多轮调整时如果 seed 没固定，每次都是从新的随机起点出发，"调"的感觉是错觉。
2. **Cross-attention 的"注意力争夺"**。加一个新词，原来某些词的关注度就被稀释了。"加一个 detail 改一处" 在 attention 机制下经常做不到。
3. **prompt 已经超过 token 上限**。SD 1.5 是 77 token，超出部分被截断。"我加了那么多描述但没用" 的根源经常在这。
4. **概念冲突**。比如同时写 "realistic" 和 "anime"，模型在两个分布间被拉扯。

### "为什么 prompt engineering 是个真技能"

整条链路有至少 5 处会损耗信息：

1. **分词（tokenization）**：你写的"赛博朋克"可能被切成 ["赛博", "朋克"] 两个 token，每个 token 单独被编码，原本的整体语义部分丢失。
2. **编码器的训练分布**：CLIP 训练时见的英文图文为主，中文 prompt + 西方文化概念的组合先天弱势。
3. **Cross-attention 的注意力分配**：模型不一定把注意力放在你认为重要的词上。
4. **CFG 的放大/抑制**：高 CFG 把某些特征放大到失真，低 CFG 把它们抹掉。
5. **采样过程的随机性**：seed + 调度器（scheduler）+ 步数共同决定了从噪声出发的具体路径。

**Prompt engineering 的本质，是用户在没看到上述 5 处黑盒的情况下，通过试错猜出"用什么词、怎么组合、加什么权重"才能让信号穿透 5 层损耗到达终点**。

这也是为什么"自然语言交互"对用户来说从来不自然——他们不是在和"懂语言的 AI"对话，他们是在和一个"靠匹配训练样本来理解的图像系统"对话。

---

## 5. 产品设计师的几个落地洞察

把上面的内容压缩成几条可以拿去做产品决策的判断：

1. **"模型理解"是分层的，前端可干预的层比想象多**。文本编码器只是其中一层；prompt rewrite、负面 prompt、CFG、ControlNet 强度、IP-Adapter scale 都是产品可以选择"暴露给用户 / 自动调"的层级。每暴露一层，自由度上升、心智负担也上升。

2. **"意图理解"和"创作能力"是两件事**。意图理解强（"它听懂了我说啥"）≠ 创作能力强（"它画得好看"）。Midjourney 的画质美学是通过特定训练数据 + RLHF 调出来的，它的意图理解未必比开源模型强。

3. **prompt rewrite 是个隐藏的产品决策**。改写后的 prompt 才是模型真正"理解"的东西。要不要让用户看见、要不要让用户编辑改写后的版本——是真实的设计岔路口。Hunyuan 直接开源了 PromptRewrite 模型，等于把这件事透明化。

4. **多输入混合时，"听谁的"有优先级**。ControlNet 优先级最高（结构是硬约束），IP-Adapter 次之（风格是软约束），prompt 最弱（语义经常被结构和风格压过去）。设计混合输入界面时，要让用户能感知和调整这个优先级。

5. **失败诊断要分层问**。不是"为什么不行"，而是"哪一层不行"——是文本理解错了？还是结构控制太强了？还是 LoRA 触发词没写对？产品如果能给出这种分层提示，能极大降低用户的挫败感。

6. **视频意图理解的下一步是 LLM 化**。HunyuanVideo 已经在做了：用 MLLM 当编码器、用专门的 LLM 做 prompt rewrite。这个方向意味着未来用户可以用更"对话式"的方式描述视频（"先慢慢推近，然后切到反应镜头"），而不是堆视觉关键词。

---

## 来源列表

**文本编码器、cross-attention、潜空间**
- The Illustrated Stable Diffusion: https://jalammar.github.io/illustrated-stable-diffusion/
- Stable Diffusion (Wikipedia): https://en.wikipedia.org/wiki/Stable_Diffusion
- Stable Diffusion 3 Research Paper: http://stability.ai/news-updates/stable-diffusion-3-research-paper
- HuggingFace Diffusion Course Unit 3: https://huggingface.co/learn/diffusion-course/en/unit3/2

**ControlNet, IP-Adapter, LoRA**
- HuggingFace Diffusers ControlNet docs: https://huggingface.co/docs/diffusers/en/using-diffusers/controlnet
- HuggingFace Diffusers IP-Adapter docs: https://huggingface.co/docs/diffusers/en/using-diffusers/ip_adapter
- HuggingFace LoRA blog: https://huggingface.co/blog/lora
- HuggingFace Diffusers img2img docs: https://huggingface.co/docs/diffusers/en/using-diffusers/img2img

**视频生成**
- Sora Review (arXiv 2402.17177): https://arxiv.org/html/2402.17177v3
- Sora Wikipedia: https://en.wikipedia.org/wiki/Sora_(text-to-video_model)
- Runway Gen-3 Alpha: https://runwayml.com/research/introducing-gen-3-alpha
- HunyuanVideo: https://github.com/Tencent/HunyuanVideo
- Wan 2.1: https://github.com/Wan-Video/Wan2.1
- FLUX (Wikipedia): https://en.wikipedia.org/wiki/Flux_(text-to-image_model)

---

## 还没搞清楚的部分（诚实标记）

- **可灵（KLING）的具体架构和文本编码器**：官网和论文公开信息都很少，无法确认它用的是 T5 系还是 LLM 系编码器。如果做相关产品决策需要更准确的对比信息，要从快手的工程师博客或访谈里找。
- **字节内部模型（豆包视觉、即梦）的意图理解链路**：这是入职后才能知道的。但从公开论文（如 PixArt-α、Seedream 系列）看，字节方向偏向于用更大的 LLM 当文本编码器 + 大量 in-house 数据 recaptioning，整体路线和 HunyuanVideo 接近。
- **Cross-attention 在 DiT 架构（替换 UNet）下的具体差异**：DiT 把 attention 模式从"局部 + cross"改成"全局 self-attention 跨所有 token"，这对意图理解的影响（比如长 prompt 是否更稳定）目前公开材料给的是定性结论，缺少对比实验。
- **多模态输入的"优先级"在不同模型里的具体表现**：本文给的是经验直觉，没有量化对比。做产品决策时如果要明确"在 A 模型里 ControlNet 多强能压过 prompt"，需要自己跑实验。
