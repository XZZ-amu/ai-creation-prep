# AI 创作准备

入职抖音 AI 创作特效团队（产品设计师）前的领域知识储备。

---

## 五个核心判断（入职前必看）

> 从 12 个 topic 的研究中提炼出的最高层结论。详见 [跨 Topic 洞察](insights/cross-topic-insights.md)。

1. **用户不写 prompt** — 抖音用户和 Midjourney 用户本质不同。交互以"选模板 + 传素材"为基线，不以文字输入为基线。
2. **混合方案是成本最优解** — AI 做一次理解/生成（花一次钱）+ 端侧渲染多次（免费）。每个特效设计时问"AI 的工作量能不能最小化"。
3. **首次成功率是最高 ROI 指标** — 它同时降低成本、提升速度感知、减少迭代、提高发布率。连锁反应比任何单点优化都大。
4. **竞争力在闭环短，不在生成强** — 抖音的优势是"从生成到社交奖励的闭环比别人短"（等待可被吸收、发布零跳转、链式反应在同一 App 内完成）。
5. **社交奖励才是驱动力** — 用户为了"发出去被看见"用特效，不是为了"做好看的东西"。KPI 应该是"使用后发布率 × 发布后互动率"。

---

## 当前进度

| 阶段 | 说明 | 能力/机制类 | 产品命题类 |
|---|---|---|---|
| ① 理解机制 | 这件事的本质是什么 | 8/10 | 4/4 |
| ② 识别问题 | 过程中常见出什么问题 | 8/10 | 4/4 |
| ③ 对比解法 | 各家怎么解的、底层逻辑 | 8/10 | 4/4 |
| ④ 抖音判断 | 在抖音特效场景下什么合适 | 8/10 | 4/4 |
| ⑤ 动手实践 | 验证解法的真实表现 | 0/10 | 0/4 |
| ⑥ 沉淀体感 | 我的看法 + 还没搞清楚的 | 0/10 | 0/4 |

---

## 各 Topic 一句话结论

### 能力 / 机制类

| # | Topic | 抖音场景的核心判断 |
|---|---|---|
| 1 | [意图理解](topics/intent-understanding/04-douyin-take.md) | 输入是"选模板+传素材"不是写 prompt，失败时系统兜底不让用户改 |
| 2 | [迭代精准性](topics/iteration-precision/04-douyin-take.md) | "不满意→换一个"覆盖 70% 需求，3 轮以内出片是产品边界 |
| 3 | [风格统一](topics/style-consistency/04-douyin-take.md) | 弱需求，"差不多像"就够，零训练方案是唯一合理路径 |
| 5 | [生成质量](topics/quality-and-fallback/04-douyin-take.md) | "好玩"比"精美"重要，75 分够用，后台 best-of-N 自动选最好 |
| 6 | [生成速度](topics/speed-and-waiting/04-douyin-take.md) | 图片 <3 秒，视频异步但可"继续刷抖音"等通知 |
| 7 | [多模态对齐](topics/multimodal-alignment/04-douyin-take.md) | 图→视频是第一优先级，冲突在模板层消灭 |
| 8 | [人脸与安全](topics/face-and-safety/04-douyin-take.md) | 60-70% 像 + 风格惊喜，安全审核 <500ms 且用户无感 |
| 10 | [算力成本](topics/compute-cost/04-douyin-take.md) | 图片可全量开放，视频必须限频，混合方案是最大杠杆 |

### 产品命题类

| # | Topic | 抖音场景的核心判断 |
|---|---|---|
| 11 | [低门槛差异化](topics/differentiation-at-low-barrier/04-douyin-take.md) | 用户素材（脸/环境/宠物）是零成本差异化来源 |
| 12 | [Feed 吸引力](topics/feed-attractiveness/04-douyin-take.md) | 叙事钩子 > 纯视觉冲击，特效应内置故事结构 |
| 13 | [发布转化](topics/publishing-conversion/04-douyin-take.md) | 一键发布 + AI 配文是最高 ROI 优化 |
| 14 | [链式反应](topics/chain-reaction/04-douyin-take.md) | 低门槛 + 高辨识度 + 可变体 = 传播公式 |

---

## 怎么读

**快速了解**：看上面的五个核心判断 + 一句话结论表就够了。

**深入某个 topic**：每个 topic 有四份文档——

- **01-mechanism** — 这件事的本质是什么
- **02-problems** — 常见失败模式和用户痛点
- **03-solutions** — 各家怎么解的、对比分析
- **04-douyin-take** — 在抖音特效场景下的产品判断

**第一次读**：建议从 [意图理解的机制文档](topics/intent-understanding/01-mechanism-v2.md) 开始——它建立了贯穿全部文档的画室比喻体系。

---

## 跨 Topic 洞察

- [跨 Topic 贯穿性结论](insights/cross-topic-insights.md) — 8 条横跨多个 topic 的元原则
- [抖音特效技术架构](insights/douyin-effects-architecture.md) — 端侧 vs 云端的架构理解

---

## 项目本身

- [产品定位](product-foundation.md) — 抖音特效产品的本质和兄弟产品边界（所有判断的锚点）
- [Topic 清单](topic-list.md) — 所有研究主题的状态
- [术语表](glossary.md) — 核心术语一句话解释
- [竞品观察](competitive/README.md) — 截图 + 结构化分析（持续采集中）
