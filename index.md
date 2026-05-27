# AI 创作准备

入职抖音 AI 创作特效团队（产品设计师）前的领域知识储备。

## 怎么读这份文档

如果你是**第一次读**：建议从 **[1. 意图理解](topics/intent-understanding/01-mechanism-v2.md)** 开始——这一篇建立了贯穿全部文档的**画室五人组比喻体系**（翻译官、画家、草稿世界、骨架图、风格手册），后续 7 份文档全部沿用这套比喻并在它基础上扩展角色。

每份机制文档（`01-mechanism.md`）都包含：

- **一句话总结** — 给非技术 PM 看的核心要点
- **贯穿案例** — 一个具体场景串起整篇
- **画室角色扩编** — 在 v2 五人组基础上新增的角色
- **机制拆解** — 每个机制都附"比喻不准在哪"和"回到案例"
- **用户视角失败现象** — 抽象机制对应的产品问题
- **产品落地洞察** — 5-6 条给产品设计师的判断
- **还没搞清楚的部分** — 诚实标记（"待入职查"）

## 文档清单

| # | Topic | 主要解决 |
|---|---|---|
| 1 | [意图理解](topics/intent-understanding/01-mechanism-v2.md) | 文字 prompt 怎么变成图——画室五人组基础范式 |
| 2 | [迭代精准性](topics/iteration-precision/01-mechanism.md) | 怎么"只改一处不动其他"——inpainting / P2P / FLUX Kontext |
| 3 | [风格统一](topics/style-consistency/01-mechanism.md) | 同一角色 / 同一风格批量出图 |
| 5 | [生成质量与兜底](topics/quality-and-fallback/01-mechanism.md) | 评分 + 修补匠 + auto-retry |
| 6 | [生成速度与等待](topics/speed-and-waiting/01-mechanism.md) | 蒸馏 + 量化 + 流式渲染 |
| 7 | [多模态对齐](topics/multimodal-alignment/01-mechanism.md) | 文字 + 图 + 音频联合控制 |
| 8 | [人脸与安全](topics/face-and-safety/01-mechanism.md) | 身份注入 + NSFW + 合规 |
| 10 | [算力成本](topics/compute-cost/01-mechanism.md) | 一次特效在后端烧多少 GPU·秒 |

> Topic 4、9 不在这一批——4（数据飞轮）和 9（具体玩家深挖）是入职后才有现实信息基础的事。

## 项目本身

- [产品定位](product-foundation.md) — 抖音特效产品的本质和兄弟产品边界（所有判断的锚点）
- [Topic 清单](topic-list.md) — 所有研究主题的状态
- [术语表](glossary.md) — 遇到时补充，不预设
