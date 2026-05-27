# 术语表

> 遇到新概念时补充。格式：术语 — 一句话解释（产品语言）。

---

## 模型与架构

| 术语 | 解释 |
|------|------|
| **Diffusion Model** | 扩散模型。通过"加噪→去噪"的过程生成图像/视频的模型范式 |
| **UNet** | U 型网络。老一代扩散模型的核心骨架（SD 1.5、SDXL），正在被 DiT 替代 |
| **DiT** (Diffusion Transformer) | 用 Transformer 替代 UNet 做去噪的新架构，支持更大参数量和更好的 scaling（FLUX、SD3） |
| **MMDiT** | Multi-Modal DiT。SD3/FLUX 用的架构，图像和文本 token 在同一个 attention 里交互 |
| **Latent Space** | 潜空间。图像被 VAE 压缩后的低维空间，模型在这里工作而非在像素空间，省算力 |
| **VAE** (Variational Autoencoder) | 变分自编码器。负责图像↔潜空间之间的压缩/解压 |
| **MoE** (Mixture of Experts) | 混合专家。每次推理只激活部分参数，降低计算量 |

## 文本理解（翻译官）

| 术语 | 解释 |
|------|------|
| **CLIP** | OpenAI 的文本-图像对齐模型。把文字和图片映射到同一个向量空间 |
| **T5** | Google 的文本编码器。比 CLIP 能处理更长、更复杂的句子（FLUX/SD3 在用） |
| **Text Encoder** | 翻译官。把用户写的人话变成模型能懂的数字矩阵 |
| **Prompt Rewrite** | 大模型自动扩写用户的简短 prompt，补充细节后再给生成模型 |
| **CFG** (Classifier-Free Guidance) | 无分类器引导。控制生成结果对 prompt 的"听话程度"，值越高越贴合但可能过饱和 |
| **Cross-Attention** | 交叉注意力。画家"读"翻译官翻译结果的机制，决定图像每个区域听哪些词 |
| **Token** | 文本被切成的最小单元。一个中文字 ≈ 1-2 个 token |

## 图像控制（骨架图 + 风格手册）

| 术语 | 解释 |
|------|------|
| **ControlNet** | 骨架图。给画家一张硬约束——按这个姿势/边缘/深度来画 |
| **LoRA** (Low-Rank Adaptation) | 低秩微调。用少量数据训练一个小插件（几十 MB），改变模型的风格/角色 |
| **IP-Adapter** | 图像提示适配器。传入参考图，让生成结果继承它的风格/内容，不需要训练 |
| **DreamBooth** | 用 3-5 张照片训练模型"记住"一个特定人物/物体/风格 |
| **Textual Inversion** | 把一个概念压缩成一个伪单词 embedding（几 KB），最轻量的个性化方案 |
| **Reference-only** | 传参考图通过共享 self-attention K/V 传递风格，不需要额外模型 |

## 编辑与迭代

| 术语 | 解释 |
|------|------|
| **Inpainting** | 局部重绘。用 mask 圈出要改的区域，区域外锁定不动 |
| **img2img** | 图生图。从一张已有图加噪后重新去噪，strength 控制改动幅度 |
| **Prompt-to-Prompt (P2P)** | 通过注入 attention map 实现不画 mask 的精准编辑（UNet 时代方案） |
| **InstructPix2Pix** | 指令式编辑。用户写"把天空变成夜晚"，模型自己判断改哪里 |
| **FLUX Kontext** | BFL 的 DiT 编辑方案。参考图作为 token 拼接输入，支持多轮编辑低漂移 |
| **Step1X-Edit** | 阶跃星辰的编辑模型。Qwen-VL 理解指令 + DiT 执行编辑 |

## 人脸与身份

| 术语 | 解释 |
|------|------|
| **ArcFace** | 人脸识别模型。把人脸压缩成 512 维向量，用于身份比对 |
| **InstantID** | 小红书+北大的人脸注入方案。单照片零训练，保真度高 |
| **PuLID** | 字节的人脸注入方案。用对比学习对齐，对非人脸区域干扰小 |
| **PhotoMaker** | 腾讯 ARC 的方案。支持多照片合并，增强身份稳定性 |
| **ConsisID** | 视频人脸一致性方案。频率分解处理，解决跨帧身份漂移 |
| **CodeFormer** | 人脸修复模型。用 codebook + transformer 修复崩坏的人脸 |

## 速度与成本

| 术语 | 解释 |
|------|------|
| **蒸馏 (Distillation)** | 把大模型的能力"压缩"进小模型/少步数模型（如 50 步→4 步） |
| **LCM** (Latent Consistency Model) | 一致性蒸馏方案，4-8 步出图 |
| **SDXL Lightning** | 字节的渐进式对抗蒸馏，1-8 步 1024px |
| **量化 (Quantization)** | 降低模型数字精度（FP16→INT8→INT4），省显存和计算量，轻微损失质量 |
| **GGUF** | 一种量化模型文件格式，适合本地/端侧部署 |
| **DeepCache** | 跨步缓存 UNet 浅层激活，降本 30-50%（对 DiT 适配差） |
| **Flash Attention** | 高效 attention 实现，减少显存占用，不影响结果 |
| **GPU·秒** | 算力消耗单位。一次图片生成约 2-10 GPU·秒，视频约 60-300 GPU·秒 |

## 质量评估

| 术语 | 解释 |
|------|------|
| **LAION Aesthetic** | 美学打分模型。CLIP + MLP，0-10 分 |
| **HPS v2** | 人类偏好打分。基于 79.8 万对人类偏好选择训练 |
| **ImageReward** | 清华/智谱的偏好模型。BLIP backbone，综合评估 prompt 匹配度+美学 |
| **VBench** | 视频质量评估框架。16 个维度（时间一致性、运动平滑性等） |
| **Best-of-N** | 一次生成 N 张，选最好的一张给用户。质量提升但成本翻 N 倍 |
| **DPO** (Direct Preference Optimization) | 直接用人类偏好数据优化模型，不需要单独训练 reward model |

## 安全与合规

| 术语 | 解释 |
|------|------|
| **NSFW** | Not Safe For Work。色情/暴力等不适当内容 |
| **Safety Checker** | 安全检查器。基于 CLIP 的后置过滤，检测生成结果是否违规 |
| **C2PA** | 内容来源与真实性联盟。给 AI 生成内容加密签名元数据 |
| **SynthID** | Google 的隐形水印技术。嵌入生成过程，不影响画面 |
| **深度合成规定** | 中国网信办法规。要求 AI 生成内容必须标识、人脸需授权 |

## 多模态

| 术语 | 解释 |
|------|------|
| **ImageBind** | Meta 的统一 embedding 空间。把文/图/音/视/深度/热力图对齐到同一空间 |
| **I2V** (Image-to-Video) | 图生视频。以首帧图片为条件，生成后续帧 |
| **Audio-driven** | 音频驱动。用声音/音乐控制人物口型、表情、动作 |
| **EMO** | 阿里的音频驱动肖像动画。单照片 + 音频 → 说话/唱歌视频 |
| **SadTalker** | 3D 运动系数驱动的说话头方案 |

## 部署工具

| 术语 | 解释 |
|------|------|
| **ComfyUI** | 节点式工作流编辑器。支持所有主流模型的本地部署和可视化 |
| **HuggingFace Diffusers** | Python 库。5 行代码跑任何扩散模型 |
| **safetensors** | 安全的模型权重文件格式。比 .pt/.ckpt 更安全（无代码执行风险） |
